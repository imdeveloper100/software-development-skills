---
name: engineering-standards
description: Baseline engineering rules for ANY code change in this repo — naming, function design, abstraction decisions, error handling, logging, config/env/secrets, dependencies, refactoring safety and backward compatibility. Use whenever writing, modifying or refactoring code, and before adding a dependency, layer or abstraction.
---

# Engineering Standards

## Purpose

Keep the codebase readable, consistent and safe to change. These rules apply to every language and layer; specialized skills add detail on top.

## Rules

### R1. Inspect first, follow conventions
- Read the surrounding code before writing new code. Match its naming, file layout, import style, error style and test style.
- If two conventions exist, follow the one used by the most recently maintained code in the same area, and mention the inconsistency.
- Do not introduce a new pattern (folder scheme, state library, error format) inside a feature task. Propose it separately.

### R2. Naming
- Names describe **intent**, not type: `activeStudents`, not `studentArray`; `isEnrollmentOpen`, not `flag`.
- Booleans: `is/has/can/should` prefix. Functions: verb first (`calculateInvoiceTotal`, `fetchStudents`). Event handlers: `handleX` inside components, `onX` for props.
- Files: follow the project. Default when none: `PascalCase.tsx` for components, `camelCase.ts` / `kebab-case.ts` for modules, `*.service.ts`, `*.controller.ts`, `*.dto.ts` on the backend.
- Constants: `UPPER_SNAKE_CASE` only for true compile-time constants (`MAX_UPLOAD_SIZE_MB`). Config objects and lists use `camelCase`.
- No abbreviations except universally known ones (`id`, `url`, `dto`, `db`). No `data2`, `tmp`, `handleClick2`.

### R3. Functions and modules
- One function = one level of abstraction. If you need comments to separate "sections" inside a function, extract them.
- Guide, not law: functions > ~40 lines or > 3 levels of nesting are a smell — check whether extraction improves clarity.
- Prefer early returns over nested `if/else`.
- Max ~3–4 positional params; beyond that, pass an options object.
- Pure functions for calculations; isolate side effects (I/O, DB, network) at the edges.
- Files > ~300 lines (components) / ~400 lines (modules) should be checked for multiple responsibilities.

### R4. Abstraction: DRY vs. premature abstraction
- **Rule of three:** duplicate once is acceptable; extract when the third occurrence appears OR when the duplicated logic is business-critical (pricing, permissions, validation) and must not drift.
- Extract only when the copies change **for the same reason**. Two code blocks that look alike but serve different domains are not duplication.
- Do not create interfaces, base classes, factories, or generic wrappers with a single implementation and no realistic second one.
- Composition over inheritance. Inheritance only for genuine "is-a" with shared behavior (e.g., framework base classes). Never more than one level of app-defined inheritance.
- SOLID is a lens, not a checklist:
  - **SRP** — apply always (one reason to change per module).
  - **DIP / interfaces** — apply at real boundaries (external services, payment, email, storage, mock-vs-real API), not between every internal class.
  - **OCP / LSP / ISP** — apply when you actually have variants (multiple payment providers, multiple notification channels).

### R5. Separation of concerns
- UI does not contain business rules that the backend must also enforce. Business rules live server-side; UI may mirror them for UX only.
- Controllers/route handlers: HTTP only. Services: business logic. Data access: queries only.
- Formatting (dates, currency) lives in shared utilities, not repeated inline.

### R6. Error handling
- Never swallow errors: no empty `catch {}`; no `catch (e) { console.log(e) }` that continues as if successful.
- Catch only where you can **handle** (retry, fallback, translate to a user/domain error) — otherwise let it propagate to the central handler.
- Throw typed/domain errors (`NotFoundError`, `ForbiddenError`, Nest `NotFoundException`) rather than generic `Error('something')`.
- Preserve the cause: `throw new PaymentFailedError('Charge failed', { cause: err })`.
- Never expose stack traces, SQL, or internal messages to clients (see `api-design` error format).
- Async: every promise is awaited or explicitly handled; no floating promises. In Express, ensure async errors reach the error middleware (Express 5, or a wrapper in Express 4).

### R7. Logging
- Use the project's logger (pino, winston, Nest `Logger`). No `console.log` left in committed backend code; frontend `console.log` only behind dev guards.
- Log levels: `error` (needs attention), `warn` (unexpected but handled), `info` (business events: user created, payment succeeded), `debug` (dev detail).
- Structured logs: `logger.info({ orderId, tenantId }, 'Order created')`, not string concatenation.
- **Never log** passwords, tokens, full card numbers, secrets, full request bodies of auth endpoints, or PII beyond IDs.
- Include a correlation/request ID where the project supports it.

### R8. Configuration, env vars, secrets
- All environment-dependent values come from config, never hardcoded (URLs, keys, feature toggles, limits).
- Validate env at startup (zod schema, Nest `ConfigModule` with `validationSchema`, `envalid`). Fail fast with a clear message when a required var is missing.
- Access env through a typed config module, not `process.env.X` scattered across files.
- Maintain `.env.example` with every variable (no real values). Update it whenever you add a variable.
- Secrets never in source, never in client bundles, never in Docker images, never in logs.
- Frontend: only variables with the public prefix (`NEXT_PUBLIC_`, `VITE_`) reach the browser — treat them as **public**. Never put secrets behind those prefixes.

### R9. Dependencies
Before adding a package, check:
1. Is it already solvable with the platform or an existing dependency? (e.g., `Intl`, `structuredClone`, `fetch`, `date-fns` already present)
2. Is it maintained (recent releases, open issue health), typed, and reasonably sized? Check bundle impact for frontend deps.
3. Does it duplicate an existing dep (second date library, second HTTP client, second icon set)? → don't add.
- Install with the project's package manager; don't create a second lockfile.
- Pin via lockfile; don't bump unrelated packages in a feature change.

### R10. Backward compatibility & refactoring safety
- Public surfaces (API responses, routes, exported functions, events, DB columns, env vars) change only with consumer search + migration path.
- Additive changes first (new field, new endpoint), removal later (deprecate → migrate → remove).
- Refactors don't change behavior. Keep refactor commits separate from behavior changes.
- Before refactoring untested critical code, add a characterization test or verify manually and describe how.
- Use search (`rg`) to find all usages before renaming or deleting anything.

### R11. Types
- TypeScript `strict` where the project allows. No `any` in new code; use `unknown` + narrowing. `as` casts only at trust boundaries with validation.
- Validate external data at the boundary (API responses in critical paths, request bodies, env) with zod/class-validator; infer types from schemas when the project does so.
- Share types between frontend and backend only through an explicit contract (shared package, OpenAPI-generated types) — not by importing backend internals into the frontend.

### R12. Comments
- Comment **why**, not what. Explain non-obvious decisions, workarounds, business rules and links to tickets.
- Delete commented-out code. Git remembers.
- `TODO` must say what and why: `// TODO(auth): remove legacy token fallback after mobile v3 rollout`.

## Implementation Guidelines

1. Locate similar existing code (`rg "createUser|class .*Service"`) and model the change on it.
2. Write the change in the smallest scope that solves the problem.
3. Name things; read the diff top-to-bottom as a reviewer would.
4. Remove dead code you created; leave unrelated dead code alone (mention it).
5. Run typecheck, lint and tests for the touched area.

## Examples

**Early return instead of nesting**
```ts
// ❌
function canEnroll(student, course) {
  if (student) {
    if (student.isActive) {
      if (course.seatsLeft > 0) return true;
    }
  }
  return false;
}

// ✅
function canEnroll(student: Student, course: Course): boolean {
  if (!student?.isActive) return false;
  return course.seatsLeft > 0;
}
```

**Typed config instead of scattered env access**
```ts
// config/env.ts
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  DATABASE_URL: z.string().url(),
  JWT_ACCESS_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(4000),
});

export const env = EnvSchema.parse(process.env); // throws at boot if invalid
```

**Error with cause, handled at the right layer**
```ts
try {
  await paymentGateway.charge(input);
} catch (err) {
  throw new PaymentFailedError('Card charge failed', { cause: err }); // domain error; central handler maps to 402/422
}
```

**Premature abstraction**
```ts
// ❌ Interface + abstract class + factory for a single email provider used in one place
interface IEmailSender { send(...): Promise<void> }
abstract class BaseEmailSender implements IEmailSender { ... }
class EmailSenderFactory { static create() { return new SesEmailSender(); } }

// ✅ One module with a clear function; introduce an interface when a second provider/mocking need appears
export async function sendEmail(to: string, template: EmailTemplate, vars: Record<string, string>) { ... }
```

## Anti-patterns

- Empty `catch` blocks, or `catch` that logs and then returns success.
- `utils.ts` / `helpers.ts` dumping grounds with unrelated functions (split by domain: `date.ts`, `currency.ts`).
- Generic `BaseService<T>` / `BaseRepository<T>` hierarchies that every module must extend.
- Magic numbers/strings repeated in multiple files (`'ADMIN'`, `86400`).
- Adding lodash for one `debounce` when the project has no lodash.
- `process.env.X!` sprinkled throughout the code.
- Renaming files/variables across the project "while you're there".
- Mixing a refactor and a feature in one change.
- Leaving `console.log`, commented-out code, or unused imports.

## Checklist

- [ ] I read the surrounding code and followed existing conventions.
- [ ] Names express intent; no unexplained abbreviations.
- [ ] No new abstraction without ≥2 real uses or a real boundary.
- [ ] No swallowed errors; domain errors used; causes preserved.
- [ ] No secrets, tokens or PII in code or logs.
- [ ] New env vars are validated and added to `.env.example`.
- [ ] No new dependency without justification; no duplicate-purpose deps.
- [ ] Public contracts unchanged, or consumers updated / migration path documented.
- [ ] No `any`, no stray `console.log`, no commented-out code.
- [ ] Typecheck, lint, and relevant tests pass.
