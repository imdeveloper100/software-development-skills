# Senior Engineer Skills for Claude Code

A reusable set of Claude Code skills plus a root `CLAUDE.md` that make Claude work like a pragmatic senior full-stack engineer on MERN, React, Next.js, NestJS, Express, PostgreSQL and MongoDB projects.

```
CLAUDE.md                              # constitution: workflow, priorities, decision rule, skill map, Definition of Done
.claude/skills/
├── engineering-standards/SKILL.md     # naming, functions, abstraction, errors, logging, config/secrets, deps, refactor safety
├── frontend-architecture/SKILL.md     # folders, components, hooks, API layer, state placement, constants/content
├── react-performance/SKILL.md         # TanStack Query/SWR, duplicate/waterfall requests, lazy loading, re-renders, lists
├── nextjs/SKILL.md                    # RSC vs client, server fetching, caching/revalidation, actions, SEO, images/fonts
├── design-system/SKILL.md             # tokens, no hardcoded colors, semantic colors, dark mode, a11y basics
├── mock-first-development/SKILL.md    # contract-first, mock vs real behind one interface, env switch
├── backend-architecture/SKILL.md      # modular monolith, thin controllers, services, errors, jobs/queues/cache decisions
├── nestjs/SKILL.md                    # modules, DTOs, ValidationPipe, guards, filters, config, Swagger, testing
├── api-design/SKILL.md                # REST conventions, slim responses, lookup endpoints, pagination, errors, idempotency
├── database/SKILL.md                  # Mongo + Postgres schema, indexes, projections, N+1, migrations, pooling
├── authentication/SKILL.md            # sessions vs JWT, refresh rotation, storage, hashing, RBAC, CSRF
├── security/SKILL.md                  # injection, XSS, IDOR, uploads, SSRF, secrets, logging, deps
├── saas-architecture/SKILL.md         # tenancy, isolation, memberships, plans/entitlements, billing, flags, audit
├── performance/SKILL.md               # measure-first method, frontend + backend fixes, caching rules
├── testing/SKILL.md                   # what/where to test, mocking boundaries, deterministic tests
├── docker/SKILL.md                    # multi-stage images, .dockerignore, secrets, health checks, compose
├── devops/SKILL.md                    # CI/CD, environments, migrations in deploys, rollback, monitoring, AWS mapping
├── system-design/SKILL.md             # requirements & estimates, component triggers, microservices criteria, trade-offs
└── code-review/SKILL.md               # 13-pass review, severity levels, concrete findings, self-review before done
templates/
├── SKILL.template.md                  # blank skill template
└── examples/console-performance/      # example project-specific specialized skill
```

## Structure decisions (vs. the original list)

- **Added `security`** as its own skill (separate from `authentication`): authn/authz is one concern; injection/uploads/SSRF/secrets is another and is needed even in code that doesn't touch auth.
- **Added `design-system`** and **`mock-first-development`**: they are key requirements and trigger on different tasks than general frontend architecture.
- **Folded in** pagination + lookup endpoints → `api-design`; constants/static content → `frontend-architecture`; error handling → `engineering-standards` (general), `backend-architecture`/`nestjs` (server), `frontend-architecture` (UI states); change safety + architectural decision rule + Definition of Done → `CLAUDE.md` (they apply to every task).

## Install

**Per project (recommended — versioned with the code):**
```bash
# from your project root
cp -r /path/to/claude-skills/.claude .
cp /path/to/claude-skills/CLAUDE.md .        # if the project already has one, merge sections instead of overwriting
git add CLAUDE.md .claude/skills && git commit -m "Add engineering skills"
```

**For all your projects (personal):**
```bash
mkdir -p ~/.claude/skills
cp -r /path/to/claude-skills/.claude/skills/* ~/.claude/skills/
cp /path/to/claude-skills/CLAUDE.md ~/.claude/CLAUDE.md   # global constitution
```
Project skills and a project `CLAUDE.md` add to the personal ones; keep project-specific rules in the project.

## How it's used

- Claude Code reads `CLAUDE.md` at session start. It tells Claude to inspect the project, pick the relevant skills from the Skill Map, verify (typecheck/lint/tests/build) and self-review.
- Each skill's `description` is always visible to Claude; the full `SKILL.md` loads only when the task matches. So 19 skills cost little context — only 2–4 load for a typical task.
- You can also ask explicitly: "use the api-design skill to review these endpoints", or invoke a skill by name as a slash command (e.g. `/code-review`).

## Adding project-specific skills

1. Copy `templates/SKILL.template.md` to `.claude/skills/<name>/SKILL.md`.
2. Write a description with concrete triggers (paths, libraries, screens, symptoms).
3. Make rules reference **real files, hooks, endpoints and numbers** in your repo.
4. Keep generic rules in the base skills; the project skill only narrows/overrides them.
5. Optional: add supporting files in the skill folder (e.g. `reference.md`, scripts) and link them from `SKILL.md`; they're read only when needed.

Good candidates: `console-performance`, `billing-module`, `report-generation`, `permissions-model`, `deployment-runbook`, `api-client-conventions`.

## Specialized skills (like `console-performance`)

See `templates/examples/console-performance/SKILL.md`. Pattern:
- **Scope** tightly: one app, module or concern (`apps/console`, the fees module).
- **Bind** generic rules to concrete code: "use `useCurrentUser()` from `features/auth/queries.ts`", "routes in `routes.tsx` use `lazy`".
- **Set budgets** you can check: bundle KB, max requests per page, p95 ms.
- **Give a procedure**: how to audit (Network tab, `rg` patterns, size script) and how to report.
- **List known anti-patterns** found in that codebase.

## How the skills avoid conflicting

- **One owner per topic.** Each rule lives in one skill; others link to it (e.g. caching decisions → `backend-architecture` R12, referenced by `performance`).
- **Layering:** `CLAUDE.md` (principles, priorities) → general skills (`engineering-standards`, `performance`) → domain skills (`api-design`, `database`) → framework skills (`nestjs`, `nextjs`) → project skills. More specific wins.
- **Existing conventions win** over skill defaults when they are reasonable (stated in every relevant skill).
- **Tie-breaker:** Correctness > Security > Maintainability > Performance > Simplicity > Scalability.
- **Decision-based rules:** skills say *when* to add Redis/queues/microservices/memoization, not *always* — so they don't push in opposite directions.
