---
name: code-review
description: Structured self-review and PR review procedure — checks a diff for regressions, architecture fit, unnecessary complexity/dependencies, API and database efficiency, duplicate/unneeded requests and renders, security, error handling, validation, type safety, accessibility, tests, and deployment impact; reports concrete, prioritized findings with file/line and fix. Use before declaring any non-trivial task complete, and whenever asked to review code, a PR, a branch, or a module.
---

# Code Review

## Purpose

Catch real problems before they ship and report them so they can be fixed quickly: specific location, impact, and concrete fix — not generic advice.

## Rules

### R1. Review the actual diff in context
- Get the diff (`git diff`, `git diff main...HEAD`, or the PR files). Read changed files fully where needed, plus callers/consumers of changed functions, endpoints, types and components (`rg` for usages).
- Review against the project's conventions and the relevant skills — not personal taste.

### R2. Findings must be concrete
Each finding = **severity · location · problem · impact · fix**.
> 🔴 **Blocker** — `students.service.ts:42` — `findUnique({ where: { id } })` has no tenant filter; any user can read another school's student by ID (IDOR). Fix: `findFirst({ where: { id, tenantId: ctx.tenantId } })` and add a cross-tenant 404 test.

Bad: "Consider improving security." / "Maybe add more tests."

### R3. Severity levels
- 🔴 **Blocker**: bugs, data loss/corruption, security/authz holes, breaking API/DB changes without migration, build/test failures, secrets exposed.
- 🟠 **Should fix**: performance problems with real impact (N+1, unbounded queries, duplicate requests, huge bundles), missing validation/error handling, missing tests for business logic, architectural drift from project patterns.
- 🟡 **Suggestion**: readability, naming, small simplifications.
- 💬 **Question**: unclear intent — ask instead of assuming.
Don't pad the review with low-value nits; if there are many style issues, the linter should handle them.

### R4. Review passes (in order)
1. **Correctness & regressions** — logic, edge cases (empty, null, boundaries, concurrency, time zones), changed behavior for existing callers, broken routes/exports/contracts.
2. **Security** — authn/authz on every new endpoint/action, tenant/ownership scoping, input validation & whitelisting, injection, XSS (`dangerouslySetInnerHTML`), secrets, sensitive logging, error leakage.
3. **Architecture** — fits existing patterns; thin controllers; logic in services; no cross-module DB access; no unjustified new layer/library/infrastructure (Architectural Decision Rule in `CLAUDE.md`).
4. **API design** — status codes, error envelope, slim responses, lookup endpoints for selectors, pagination & limits, backward compatibility.
5. **Database** — projections, N+1, indexes justified, transactions where needed, migrations safe (expand/contract).
6. **Frontend data & rendering** — server state via query library (no `useEffect` fetching), key consistency, duplicate/waterfall requests, narrow invalidation, lazy loading of heavy routes/libs, unnecessary re-renders or pointless memoization, list virtualization/pagination.
7. **UX states & accessibility** — loading/error/empty/403/404 states; semantic elements, labels, keyboard access, focus visible, alt text, color not the only signal; design tokens used.
8. **Error handling & logging** — no swallowed errors, typed errors, consistent responses, structured logs without secrets.
9. **Type safety** — no new `any`, unsafe casts, or unchecked external data.
10. **Tests** — business logic, authz and bug fixes covered; tests meaningful and deterministic.
11. **Dependencies** — new packages justified, not duplicating existing ones, lockfile updated correctly.
12. **Delivery** — env vars added to `.env.example` and config validation; Docker/CI still valid; migrations included; build passes.
13. **Maintainability** — duplication, dead code, giant files/functions, naming, leftover `console.log`/TODOs/commented code, unnecessary abstractions.

### R5. Verify, don't guess
Run typecheck, lint, tests and build when possible. If you claim "this causes duplicate requests" or "this query is slow", point to the code path (and evidence if available). If unsure, mark as 💬 Question.

### R6. Self-review before finishing a task
Before reporting a task as done, run passes 1–3 and those relevant to the change, fix blockers/should-fix items yourself, and list any remaining known issues or deliberate trade-offs in the final report.

### R7. Acknowledge what's good (briefly)
One line noting solid decisions helps calibrate — no flattery paragraphs.

## Implementation Guidelines — review output format

```
## Review: <scope>

**Summary:** <1–3 sentences: overall assessment and merge recommendation>

### 🔴 Blockers
1. `path/file.ts:LINE` — problem → impact → fix

### 🟠 Should fix
...

### 🟡 Suggestions
...

### 💬 Questions
...

**Verified:** typecheck ✅ lint ✅ tests ✅ (12 passed) build ⚠️ not run (reason)
```

## Examples

> 🟠 **Should fix** — `layouts/DashboardLayout.tsx:18` and `pages/Dashboard.tsx:9` — both call `useQuery({ queryKey: ['me'], ... })` vs `['currentUser']` with the same `/auth/me` endpoint → two requests on every dashboard load and two cache entries that can disagree. Fix: use `useCurrentUser()` from `features/auth/queries.ts` (key `authKeys.me`) in both.

> 🟠 **Should fix** — `ClassSelect.tsx:12` fetches `GET /classes` (full objects incl. `students[]`, ~400 KB) to populate a dropdown. Fix: add `GET /classes/lookup?search=&limit=` returning `{ id, name }` and use a debounced query.

> 🔴 **Blocker** — `migrations/20261003_rename_name.sql` renames `students.name` → `full_name` in the same release as the code change → running instances crash during rolling deploy. Fix: add `full_name`, backfill, read/write both, drop `name` in a later release.

> 🟡 **Suggestion** — `InvoiceRow.tsx:30` wraps `formatAmount(amount)` in `useMemo`; it's a cheap call. Remove the memo for clarity.

## Anti-patterns

- Generic comments without location or fix.
- Rewriting the author's code to personal style.
- Approving without running or reading callers of changed contracts.
- Burying blockers among dozens of nits.
- Requesting new architecture (microservices, Redis) without a demonstrated need.
- Claiming checks passed without running them.

## Checklist

- [ ] Diff and affected consumers read.
- [ ] All 13 passes considered; irrelevant ones skipped consciously.
- [ ] Every finding has severity, location, impact and fix.
- [ ] Blockers fixed (self-review) or clearly flagged (PR review).
- [ ] Typecheck/lint/tests/build run, results stated honestly.
- [ ] Remaining risks and trade-offs listed.
