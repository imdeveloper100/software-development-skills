---
name: testing
description: Testing strategy and rules for React/Next.js frontends and Node/Express/NestJS backends — what to test (business logic, critical flows, auth/authz, validation, API contracts, edge cases), which level (unit, component, integration, API/e2e), tools (Vitest/Jest, Testing Library, MSW, Supertest, Testcontainers, Playwright), test data, mocking boundaries and avoiding meaningless coverage tests. Use when writing, fixing or reviewing tests, or when deciding how to verify a change.
---

# Testing

## Purpose

Tests that catch real regressions in behavior users and the business care about, run fast, and don't break on harmless refactors.

## Rules

### R1. Use the project's test stack
Detect runner and libraries from `package.json` (Vitest/Jest, Testing Library, Playwright/Cypress, Supertest). Match existing file naming (`*.test.ts` vs `*.spec.ts`, colocated vs `__tests__/` vs `test/`). Don't introduce a second runner.

### R2. What to test (priority order)
1. Business rules (pricing, fees, eligibility, state transitions, limits).
2. Authorization: 401 unauthenticated, 403 missing permission, 404 other tenant/owner.
3. Validation: invalid input rejected with correct error shape; server-owned fields ignored.
4. Critical user flows (login, checkout/payment, create/edit core entities).
5. API contracts (response shape, status codes, pagination meta).
6. Edge cases: empty lists, boundaries (limit max, zero, negative), duplicates (409), concurrency-sensitive paths, time zones/dates.
7. Bug fixes: every bug fix gets a test that fails before the fix.

### R3. What NOT to test
- Framework/library behavior (that React renders, that Prisma saves).
- Trivial getters, constants, pure pass-through code.
- Implementation details: internal state, private methods, CSS class names, call order of internal functions.
- Snapshot tests of large component trees (brittle; nobody reviews them). Small, intentional inline snapshots are OK.
Coverage is a signal, not a target. Never write assertion-free tests to raise coverage.

### R4. Choose the level
| Code | Best level |
|---|---|
| Pure functions/business calc | Unit |
| Service with business logic | Unit with mocked boundaries (DB repo, external APIs) **or** integration with a real test DB if logic is query-heavy |
| Controller/route + validation + guards | API/integration test (Supertest / Nest e2e) — not controller unit tests that mock everything |
| DB queries, migrations, transactions | Integration against a real DB (Testcontainers, docker-compose test DB, or `mongodb-memory-server`) — never SQLite as a stand-in for Postgres semantics |
| React component with logic | Component test (Testing Library + MSW for API) |
| Hooks with logic | `renderHook` with a fresh `QueryClient` wrapper |
| Cross-page critical flows | E2E (Playwright) — few, stable, critical only |

Rough shape: many unit + integration, fewer component, few E2E.

### R5. Mock at boundaries only
Mock: external HTTP APIs (MSW / nock), email/SMS/payment providers, time (`vi.useFakeTimers`), randomness. Don't mock: your own modules under test, the ORM in integration tests, React internals. In frontend tests mock the **network** (MSW), not the API module, so the real client/query code is exercised.

### R6. Frontend testing rules
- Query by role/label/text as a user would (`getByRole('button', { name: /save/i })`); avoid `data-testid` unless no accessible handle exists.
- Use `userEvent` over `fireEvent`.
- Test loading, error, empty and success states for data views.
- New `QueryClient` per test with `retry: false`.
- Assert on visible outcome (text appears, request sent with right payload), not on state variables.

### R7. Backend testing rules
- Each test sets up its own data via factories/builders (`buildStudent({ status: 'active' })`), not shared mutable fixtures.
- Isolate DB state: truncate between tests or wrap in transactions rolled back; unique tenant per test is a simple isolation trick.
- Auth in API tests: create real tokens via a test helper, not by disabling guards — unless auth isn't the subject and the guard is explicitly overridden.
- Assert status code **and** body shape (including error `code`).

### R8. Tests must be deterministic & fast
No real network, no sleeps (use fake timers / `findBy*` / `waitFor`), fixed dates, seeded randomness. Unit suites run in seconds. Flaky tests are bugs — fix or quarantine with a ticket.

### R9. Naming & structure
`describe('<unit>')` → `it('<does what> when <condition>')`. Arrange–Act–Assert, one behavior per test. Tests read like specifications.

### R10. Run tests before declaring done
Run the relevant tests (touched modules) and report results. If you couldn't run them, say so explicitly.

## Implementation Guidelines

1. Identify behavior changed/added and its risk.
2. Pick level per R4; write the failing test for bug fixes first.
3. Use factories/MSW/test DB per project conventions.
4. Cover happy path + the relevant R2 categories (authz, validation, edge cases).
5. Run, ensure green, check no flakiness (run twice for async-heavy tests).

## Examples

**Service unit test (business rule)**
```ts
describe('FeesService.calculateDue', () => {
  it('applies sibling discount only to the second child onwards', () => {
    const due = calculateDue([{ base: 10_000 }, { base: 10_000 }], { siblingDiscountPct: 10 });
    expect(due).toEqual([10_000, 9_000]);
  });
});
```

**API test (authz + tenant isolation)**
```ts
it('returns 404 when reading a student from another tenant', async () => {
  const other = await createTenantWithStudent();
  const token = await loginAs(await createUser({ permissions: ['students.read'] }));
  await request(app.getHttpServer())
    .get(`/api/v1/students/${other.student.id}`)
    .set('Authorization', `Bearer ${token}`)
    .expect(404);
});
```

**Component test with MSW**
```tsx
it('shows empty state when there are no students', async () => {
  server.use(http.get('*/students', () => HttpResponse.json({ items: [], meta: { page: 1, limit: 20, total: 0, hasNext: false } })));
  renderWithProviders(<StudentsTable />);
  expect(await screen.findByText(/no students yet/i)).toBeInTheDocument();
});
```

## Anti-patterns

- Tests asserting that a mocked function was called with what the mock was set up to return (testing the mock).
- Controller unit tests that mock the service and only check pass-through.
- Huge snapshot files.
- `setTimeout`/`sleep` in tests.
- Shared seed data mutated by multiple tests.
- SQLite in-memory replacing Postgres for query tests.
- Disabling auth globally in test setup.
- Writing tests after the fact only to hit a coverage %.

## Checklist

- [ ] Uses existing runner/conventions.
- [ ] Business rules, authz (401/403/404), validation and critical flows covered for the change.
- [ ] Bug fixes have a regression test.
- [ ] Right test level; mocks only at boundaries.
- [ ] Deterministic (no real network, no sleeps, fixed time).
- [ ] Assertions on observable behavior and API contract.
- [ ] Tests run and pass (or explicitly reported as not run).
