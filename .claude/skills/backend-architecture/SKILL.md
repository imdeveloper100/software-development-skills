---
name: backend-architecture
description: Layering and structure rules for Node.js backends (Express, NestJS, Fastify) — modular monolith organization, thin controllers, DTO validation, services for business logic, data-access layer, middleware/guards/interceptors, centralized error handling, logging, config, transactions, background jobs, queues and caching decisions. Use when creating or changing backend modules, endpoints, services, jobs, or deciding whether to add Redis/queues/repositories.
---

# Backend Architecture

## Purpose

Keep backends predictable: every request flows through the same layers, business rules live in one place, and infrastructure (cache, queues) appears only when a real need exists.

## Rules

### R1. Modular monolith by default
Organize by **domain module**, not by technical layer across the whole app:
```
src/
├── modules/
│   ├── students/
│   │   ├── students.controller.ts   (or students.routes.ts in Express)
│   │   ├── students.service.ts
│   │   ├── students.repository.ts   (only if justified — R5)
│   │   ├── dto/ (create-student.dto.ts, student-response.dto.ts, list-students.query.ts)
│   │   ├── students.schema.ts / entities/
│   │   └── students.module.ts (Nest) / index.ts
│   ├── fees/
│   └── auth/
├── common/      # errors, middleware, guards, interceptors, pagination utils, decorators
├── config/      # typed, validated config
├── infra/       # db client, mailer, storage, queue clients
└── main.ts / app.ts
```
- A module talks to another module through its **service's public methods**, never by querying the other module's tables/collections directly. This keeps a clean seam for future extraction.
- Avoid circular module dependencies; if two modules need each other, extract the shared concept or use domain events (in-process) where justified.

### R2. Request flow
```
Route/Controller → (auth guard/middleware) → validation (DTO/schema) → Service → Data access → DB
                                                                        ↓
                                                    Response mapping (response DTO)
```

### R3. Controllers are thin
Controllers/route handlers do only: read params/body/query (already validated), get the current user/tenant, call **one** service method, return the result with the right status code. No DB queries, no business branching, no try/catch for normal errors (the central handler does that). Target < ~10 lines per handler.

### R4. Validation at the edge
- Every input (body, query, params, headers you rely on) is validated: class-validator DTOs + `ValidationPipe` (Nest) or zod/Joi middleware (Express).
- Whitelist: strip/reject unknown fields (`whitelist: true, forbidNonWhitelisted: true`) to prevent mass assignment (`role: 'admin'` in a profile update).
- Coerce/transform query params (page, limit) to numbers with bounds.
- Services may assume validated shapes but still enforce **business** invariants (e.g., seats available, state transitions).

### R5. Services hold business logic; data access is isolated
- Service methods are named by use case: `enrollStudent`, `issueInvoice`, `cancelSubscription` — not `update`.
- Services don't know about `req`/`res`.
- Data access: use the ORM/ODM directly in the service **or** a repository, following the project. Introduce a repository when at least one is true: complex queries reused across services; the service would otherwise be dominated by query code; you need to swap/mocks data sources; raw SQL/aggregation pipelines that deserve a name. Don't wrap Prisma/Mongoose in a pass-through `BaseRepository` that adds nothing.
- Never return ORM documents/entities with sensitive fields (password hashes, internal flags) directly — map to response DTOs or use explicit `select`/projection.

### R6. Response shaping
Return response DTOs / explicit selects. Lists: `{ items, meta }` (see `api-design`). Never `res.json(await Model.find())`.

### R7. Centralized error handling
- Throw domain/HTTP exceptions in services (`NotFoundException`, or custom `AppError` subclasses with `statusCode` + `code`).
- One global error handler/exception filter maps errors to the standard error response, logs 5xx with stack + request ID, and hides internals in production.
- Map known DB errors centrally (unique violation → 409, FK violation → 409/422, invalid ObjectId → 400/404).
- Express 4: wrap async handlers or use a helper so rejections reach the error middleware. Express 5 handles async errors natively.

### R8. Cross-cutting concerns — use the right hook
| Concern | Express | NestJS |
|---|---|---|
| Authentication | middleware | Guard (`JwtAuthGuard`) |
| Authorization/RBAC | middleware/route-level check + service checks for ownership | Guard + `@Roles/@Permissions` decorator; ownership in service |
| Validation | zod/Joi middleware | `ValidationPipe` + DTOs |
| Logging/request ID | pino-http middleware | middleware/interceptor |
| Response transform/timing | middleware | Interceptor |
| Error mapping | error middleware | Exception filter |

### R9. Logging & config
- Structured logger with request ID; log business events at `info`, failures at `error` with context (no secrets/PII). See `engineering-standards` R7–R8.
- Typed, validated config module; services receive config via DI/import, not `process.env`.

### R10. Transactions
Use a transaction when a use case writes **multiple** records that must succeed or fail together (create invoice + line items + ledger entry; transfer seats). Keep transactions short: no external HTTP calls or emails inside. Side effects after commit (outbox pattern if they must be reliable).
- Postgres: Prisma `$transaction`, TypeORM `DataSource.transaction`, Knex `trx`.
- MongoDB: sessions require a replica set; prefer single-document atomic updates (`$inc`, `findOneAndUpdate` with conditions) when the model allows.

### R11. Background jobs & queues — decision rule
Run work **inline** unless one is true:
- It takes > ~1–2 s or calls slow/unreliable third parties (email, PDF generation, image processing, webhooks, imports).
- It must retry with backoff independently of the user request.
- It must be scheduled/recurring.
- Load spikes would otherwise exhaust request capacity.

Then choose the lightest tool that fits: in-process `setImmediate`/fire-and-forget for non-critical, loss-tolerant work (log failures); a cron (`@nestjs/schedule`, node-cron) for scheduled jobs in a single instance; **BullMQ + Redis** (or SQS on AWS) when you need persistence, retries, concurrency control or multiple instances. Jobs must be **idempotent** (safe to run twice) and carry IDs, not full objects.

### R12. Caching — decision rule
Cache only with evidence (slow query/endpoint, hot read path, expensive third-party call) and a clear invalidation strategy.
1. First fix the query (index, projection, pagination) — see `database`.
2. HTTP caching (`Cache-Control`, ETag) for public/semi-static GETs.
3. In-memory LRU for tiny, per-instance, rarely-changing data (config, permissions map) — accept per-instance staleness.
4. Redis when multiple instances must share cache, or for rate limiting/sessions.
Always: namespaced keys including tenant (`t:{tenantId}:students:lookup:{hash}`), TTL, invalidation on writes, and never cache per-user authorized data under a shared key.

### R13. Background evolution path
Write modules so they could be extracted later: clear module APIs, no cross-module DB access, IDs instead of shared object graphs, config-injected clients. Do **not** split into services until `system-design` criteria are met.

## Implementation Guidelines

1. Find the existing module pattern and copy its shape.
2. Define DTOs (request + response) first.
3. Implement service use case with business checks + authorization-on-ownership.
4. Add data access with projections; transaction if multi-write.
5. Wire a thin controller; set status codes.
6. Ensure errors flow to the global handler; add tests for the service (see `testing`).

## Examples

**Express — thin route, validated input, service, error flow**
```ts
// students.routes.ts
router.get('/', requireAuth, validate({ query: ListStudentsQuery }), async (req, res) => {
  const result = await studentsService.list(req.auth.tenantId, req.query);
  res.json(result);
});

router.post('/', requireAuth, requirePermission('students.create'), validate({ body: CreateStudentBody }),
  async (req, res) => {
    const student = await studentsService.create(req.auth, req.body);
    res.status(201).json(student);
  });

// students.service.ts
export async function create(auth: AuthContext, input: CreateStudentInput): Promise<StudentResponse> {
  const cls = await classesService.getForTenant(auth.tenantId, input.classId); // throws NotFoundError
  if (cls.seatsLeft <= 0) throw new ConflictError('CLASS_FULL', 'Class has no seats left');
  const student = await db.student.create({
    data: { ...input, tenantId: auth.tenantId, createdById: auth.userId },
    select: studentResponseSelect,
  });
  logger.info({ studentId: student.id, tenantId: auth.tenantId }, 'Student created');
  return student;
}
```

**Express error middleware**
```ts
app.use((err, req, res, _next) => {
  const known = err instanceof AppError;
  const status = known ? err.statusCode : 500;
  if (status >= 500) req.log.error({ err }, 'Unhandled error');
  res.status(status).json({
    error: {
      code: known ? err.code : 'INTERNAL_ERROR',
      message: known ? err.message : 'Something went wrong',
      details: known ? err.details : undefined,
      requestId: req.id,
    },
  });
});
```

## Anti-patterns

- Fat controllers with queries, loops and business rules.
- `res.json(user)` where `user` includes `passwordHash`.
- Business logic duplicated across controllers or between REST and job handlers (put it in the service; both call it).
- Module A querying module B's collection directly.
- try/catch in every controller returning ad-hoc `{ success: false, msg }`.
- Sending emails/HTTP calls inside DB transactions.
- Adding BullMQ/Redis for a job that runs once a day on one instance.
- Caching without invalidation or without tenant in the key.
- Generic `BaseController<T>`/`BaseService<T>` CRUD hierarchies that hide behavior.

## Checklist

- [ ] Code placed in the right domain module; no cross-module DB access.
- [ ] Controllers thin; one service call per handler.
- [ ] All inputs validated with whitelist; response DTO/select used.
- [ ] Business rules + ownership checks in services.
- [ ] Errors thrown as typed exceptions; global handler formats them.
- [ ] Multi-write use cases transactional; no external calls inside.
- [ ] Jobs/queues/cache only with stated justification; idempotent jobs; tenant-aware cache keys.
- [ ] Structured logging with context; no secrets logged.
