---
name: nestjs
description: NestJS-specific conventions — feature modules and boundaries, controllers/services/providers, DTOs with class-validator and global ValidationPipe, guards (JWT, roles/permissions), interceptors, exception filters, middleware, custom decorators (@CurrentUser), ConfigModule with validation, Swagger/OpenAPI, database integration (Prisma/TypeORM/Mongoose) and testing with TestingModule. Use for any change in a NestJS backend.
---

# NestJS

## Purpose

Use Nest's building blocks for what they were designed for, keep modules small and domain-shaped, and get validation, auth, errors and docs consistent across the API. Applies on top of `backend-architecture`.

## Rules

### R1. Modules = domain boundaries
- One module per domain (`StudentsModule`, `FeesModule`, `AuthModule`). Split a module when it has > ~2 controllers or its service exceeds ~400 lines with unrelated use cases.
- Export only what other modules need (usually the service). Never export repositories/models for other modules to query.
- `CommonModule`/`SharedModule` only for genuinely cross-cutting providers; avoid making it a dumping ground. Use `@Global()` sparingly (config, logger, prisma).
- Resolve circular deps by redesign; `forwardRef` is a last resort and should be noted.

### R2. Bootstrap essentials (main.ts)
```ts
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(Logger));                    // nestjs-pino or custom
app.setGlobalPrefix('api');
app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });
app.useGlobalPipes(new ValidationPipe({
  whitelist: true, forbidNonWhitelisted: true, transform: true,
  transformOptions: { enableImplicitConversion: false },
}));
app.useGlobalFilters(new AllExceptionsFilter(app.get(HttpAdapterHost)));
app.enableCors({ origin: config.corsOrigins, credentials: true });
app.use(helmet());
app.enableShutdownHooks();
```
Prefer registering global guards/filters/interceptors via `APP_GUARD`/`APP_FILTER`/`APP_INTERCEPTOR` providers when they need DI.

### R3. Controllers
- Thin. Decorators for routes, `@HttpCode` where the default is wrong (e.g., 204 for delete, 200 for POST actions like `/login`).
- Use `ParseUUIDPipe`/custom `ParseObjectIdPipe` on id params.
- Get user via a `@CurrentUser()` decorator, not `@Req()`.
- Return response DTOs; avoid `@Res()` (it disables Nest's response handling) unless streaming.

### R4. DTOs
- Request DTOs with class-validator + class-transformer: `CreateStudentDto`, `UpdateStudentDto extends PartialType(CreateStudentDto)`, `ListStudentsQueryDto extends PaginationQueryDto`.
- Bound everything: `@MaxLength`, `@Max(100)` on `limit`, `@IsEnum`, `@IsUUID`/`@IsMongoId`.
- Never accept `tenantId`, `role`, `ownerId`, `createdBy` from the client body — take them from the authenticated context.
- Response DTOs (or explicit selects) define what leaves the API. If using `ClassSerializerInterceptor`, mark sensitive fields with `@Exclude()` — but explicit selects/mapping are safer than relying on exclusion.

### R5. Pipes, guards, interceptors, filters, middleware — when to use which
| Need | Use |
|---|---|
| Validate/transform input | Pipe (`ValidationPipe`, `ParseUUIDPipe`) |
| Is the user authenticated / allowed to call this route? | Guard (`JwtAuthGuard`, `PermissionsGuard`) |
| Wrap execution: logging timing, response mapping, caching, timeouts | Interceptor |
| Map exceptions to HTTP responses | Exception filter |
| Raw request concerns before routing (request ID, raw body for webhooks) | Middleware |
Ownership checks (can user X edit student Y?) belong in the **service**, which has the data — not in a guard that would need to load the entity twice.

### R6. Auth & RBAC
- Global `JwtAuthGuard` via `APP_GUARD`; opt out with `@Public()`.
- `@RequirePermissions('students.create')` + `PermissionsGuard` reading metadata with `Reflector`. Prefer permissions over raw roles in checks; map roles → permissions in one place.
- Tenant context from the token, exposed via `@CurrentUser()` / a request-scoped context; see `saas-architecture`.

### R7. Errors
- Throw built-in HTTP exceptions (`NotFoundException`, `ConflictException`, `ForbiddenException`, `UnprocessableEntityException`) or domain errors mapped by a filter.
- Include a machine-readable `code`: `throw new ConflictException({ code: 'CLASS_FULL', message: 'Class has no seats left' })`.
- One `AllExceptionsFilter` formats the standard error envelope (`api-design` R9), formats `ValidationPipe` errors into `details[]`, maps Prisma/Mongo errors (P2002 → 409, CastError → 400), logs 5xx, hides internals.

### R8. Config
`ConfigModule.forRoot({ isGlobal: true, validate: (env) => EnvSchema.parse(env) })` (zod) or Joi `validationSchema`. Use namespaced typed config (`registerAs('auth', ...)` + `ConfigType<typeof authConfig>`). No `process.env` in services.

### R9. Database integration
- Follow the project's ORM. Prisma: a `PrismaService extends PrismaClient implements OnModuleInit` in a global `PrismaModule`. TypeORM: `TypeOrmModule.forRootAsync` with `autoLoadEntities`, **`synchronize: false`** outside local dev, migrations committed. Mongoose: `MongooseModule.forFeature` per module, schemas in the module.
- Always `select`/projection for lists; transactions per `backend-architecture` R10.

### R10. Swagger/OpenAPI
If `@nestjs/swagger` is present (recommend it for APIs consumed by a separate frontend): enable the CLI plugin in `nest-cli.json` to reduce decorator noise; document response DTOs (`@ApiOkResponse({ type: StudentResponseDto })`), auth (`@ApiBearerAuth()`), and error responses for key endpoints. Disable or protect `/docs` in production if the API is private. Generated OpenAPI can feed frontend type generation.

### R11. Testing
- Unit test services with `Test.createTestingModule` and mocked dependencies (`useValue`).
- e2e tests with `supertest` against `app.getHttpServer()` using a real test database (Testcontainers or a dedicated DB), covering auth, validation (400), authz (403), and happy path.
- Override guards in tests with `.overrideGuard(JwtAuthGuard).useValue(...)` only where auth isn't under test.

### R12. Scope & performance
- Providers are singletons by default — keep them stateless. Avoid `Scope.REQUEST` unless necessary (it propagates and costs per-request instantiation); prefer passing context explicitly or using `AsyncLocalStorage` (e.g., `nestjs-cls`).
- Use the Fastify adapter only if already chosen or a measured need exists.

## Implementation Guidelines

1. `nest g module|controller|service <name>` or copy an existing module's structure.
2. DTOs → service use cases → controller → register in module → export service if needed.
3. Add guards/decorators for authz; ownership checks in service.
4. Swagger decorators for public contracts.
5. Unit test the service; e2e for critical endpoints.

## Examples

```ts
@ApiTags('students')
@ApiBearerAuth()
@Controller('students')
export class StudentsController {
  constructor(private readonly students: StudentsService) {}

  @Get()
  @RequirePermissions('students.read')
  list(@CurrentUser() user: AuthUser, @Query() query: ListStudentsQueryDto) {
    return this.students.list(user.tenantId, query);
  }

  @Get('lookup')
  @RequirePermissions('students.read')
  lookup(@CurrentUser() user: AuthUser, @Query() query: LookupQueryDto): Promise<LookupItemDto[]> {
    return this.students.lookup(user.tenantId, query);
  }

  @Post()
  @RequirePermissions('students.create')
  create(@CurrentUser() user: AuthUser, @Body() dto: CreateStudentDto): Promise<StudentResponseDto> {
    return this.students.create(user, dto);
  }

  @Delete(':id')
  @HttpCode(204)
  @RequirePermissions('students.delete')
  remove(@CurrentUser() user: AuthUser, @Param('id', ParseUUIDPipe) id: string) {
    return this.students.remove(user, id);
  }
}

export class PaginationQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) page = 1;
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(100) limit = 20;
}

export const CurrentUser = createParamDecorator(
  (_: unknown, ctx: ExecutionContext) => ctx.switchToHttp().getRequest().user as AuthUser,
);
```

## Anti-patterns

- A single `AppModule` with 30 providers; or one `AdminModule` containing every domain.
- `@Res()` + manual `res.json()` everywhere.
- `@Body() body: any` or DTOs without validators.
- `tenantId` accepted from request body.
- Guards that load entities from DB for ownership checks of every route.
- `synchronize: true` in production.
- try/catch in controllers returning custom error objects instead of throwing exceptions.
- Request-scoped providers used just to read the current user.

## Checklist

- [ ] Module per domain; only services exported; no circular deps.
- [ ] Global ValidationPipe (whitelist, forbidNonWhitelisted, transform); DTOs fully bounded.
- [ ] Global auth guard + permission decorators; ownership in services.
- [ ] Exceptions thrown with codes; global filter formats them.
- [ ] Config validated at boot; no `process.env` in services.
- [ ] Lists use selects/projections; migrations instead of synchronize.
- [ ] Swagger documents new/changed endpoints (if Swagger used).
- [ ] Service unit tests and e2e tests for critical routes pass.
