---
name: mock-first-development
description: How to build frontend features before the backend API exists — define the API contract and types first, put mock and real implementations behind one stable service interface (or MSW), switch by environment flag, keep mocks out of production builds, and swap to the real API with minimal UI change. Use when the frontend needs an endpoint that doesn't exist yet, when prototyping UI, or when replacing mocks with real integration.
---

# Mock-First Development

## Purpose

Never block frontend work on a missing endpoint, and never let mock data leak into UI code or production. The UI depends on a **contract**, not on where data comes from.

## Rules

### R1. Contract first
Before any mock, write the contract: request params, response DTO types, error shape, pagination meta — following `api-design`. Put it where the real one will live (`features/x/api/types.ts`, shared contract package, or an OpenAPI draft). Share it with the backend side if you own both.

### R2. One interface, two implementations — choose ONE mechanism per project
Check if the project already has a mocking approach and use it. Otherwise pick:

**Option A — Network-level mocking with MSW (preferred when available/acceptable).**
UI and API modules are 100% real code; MSW intercepts HTTP in dev/tests. Switching to the real backend = turning MSW off. Best fidelity, also reused in tests.

**Option B — Service-level adapter** (no extra dependency):
```
features/students/api/
├── students.contract.ts   # types + StudentsApi interface
├── students.http.ts       # real implementation using apiClient
├── students.mock.ts       # mock implementation (in-memory, latency, errors)
├── students.fixtures.ts   # mock data
└── index.ts               # selects implementation by config
```

### R3. UI never imports mocks
Components and hooks import from `features/students/api` (index) or call query hooks. A grep for `.mock` / `fixtures` in `components/` and `app/` must return nothing.

### R4. Explicit, environment-controlled switch
- Flag: `VITE_USE_MOCKS=true` / `NEXT_PUBLIC_USE_MOCKS=true` (public flag is fine — it's not a secret), default **false**.
- Per-resource override is allowed while endpoints come online gradually (`MOCK_RESOURCES=students,fees`).
- Production: the flag must be false; enforce with a startup assertion and keep mock modules out of the bundle via dynamic import inside the flag check (so they tree-shake away).

### R5. Mocks behave like the real API
- Same types, same pagination meta, same error shape.
- Simulate latency (200–600 ms) and allow forcing errors (e.g., `?mockError=500`, or a toggle) so loading/error/empty states get built.
- Respect params (search, page, limit, sort) roughly — enough to exercise UI behavior.
- Mutations update in-memory state so the UI flow (create → list refresh) works.
- Deterministic fixtures (fixed IDs / seeded faker) so screenshots and tests are stable.

### R6. Lifecycle — mocks are temporary
- Track each mocked endpoint: `// MOCK: GET /students/lookup — pending backend (ticket #123)`.
- When the real endpoint lands: switch, verify the contract (field names, nullability, dates, pagination), delete the mock for that resource or move it to test fixtures. Leaving dead mocks is a code-review finding.
- If the real API differs from the contract, adapt in the API layer (mapping), not in components — and tell the backend team if the contract was agreed.

## Implementation Guidelines

1. Write the contract types and interface.
2. Implement the mock (MSW handler or `*.mock.ts`) with fixtures, latency and error toggle.
3. Write query hooks against the interface; build UI incl. loading/error/empty.
4. Implement `*.http.ts` (can be done immediately against the agreed contract).
5. When backend is ready: flip flag, test, delete mock.

## Examples

**Option B — adapter**
```ts
// students.contract.ts
export interface StudentLookupItem { id: string; name: string }
export interface StudentsApi {
  list(params: StudentListParams): Promise<Paginated<StudentListItem>>;
  lookup(params: { search?: string; limit?: number }): Promise<StudentLookupItem[]>;
  create(dto: CreateStudentDto): Promise<StudentDetail>;
}

// students.http.ts
export const studentsHttp: StudentsApi = {
  list: (params) => apiClient.get('/students', { params }).then((r) => r.data),
  lookup: (params) => apiClient.get('/students/lookup', { params }).then((r) => r.data),
  create: (dto) => apiClient.post('/students', dto).then((r) => r.data),
};

// students.mock.ts
import { STUDENT_FIXTURES } from './students.fixtures';
const db = [...STUDENT_FIXTURES];
const delay = (ms = 300) => new Promise((r) => setTimeout(r, ms));

export const studentsMock: StudentsApi = {
  async list({ page = 1, limit = 20, search = '' }) {
    await delay();
    const filtered = db.filter((s) => s.name.toLowerCase().includes(search.toLowerCase()));
    const items = filtered.slice((page - 1) * limit, page * limit);
    return { items, meta: { page, limit, total: filtered.length, hasNext: page * limit < filtered.length } };
  },
  async lookup({ search = '', limit = 20 }) {
    await delay(150);
    return db.filter((s) => s.name.toLowerCase().includes(search.toLowerCase()))
             .slice(0, limit).map(({ id, name }) => ({ id, name }));
  },
  async create(dto) { await delay(); const s = { id: crypto.randomUUID(), ...dto }; db.unshift(s); return s; },
};

// index.ts
import { appConfig } from '@/config/app';
import { studentsHttp } from './students.http';

export const studentsApi: StudentsApi = appConfig.useMocks
  ? (await import('./students.mock')).studentsMock   // or a lazy proxy if top-level await is unavailable
  : studentsHttp;
```

**Option A — MSW**
```ts
// mocks/handlers/students.ts
export const studentHandlers = [
  http.get('/api/students/lookup', async ({ request }) => {
    await delay(150);
    const search = new URL(request.url).searchParams.get('search') ?? '';
    return HttpResponse.json(lookup(search));
  }),
];
// main.tsx
if (import.meta.env.DEV && import.meta.env.VITE_USE_MOCKS === 'true') {
  const { worker } = await import('./mocks/browser');
  await worker.start({ onUnhandledRequest: 'bypass' });
}
```

## Anti-patterns

- `const students = [{...}, {...}]` hardcoded inside a page component "for now".
- Components with `if (USE_MOCKS) ... else ...` branches.
- Mock response shapes that differ from the real contract (e.g., no pagination meta).
- Mock flag defaulting to true, or mock code shipped in the production bundle.
- Mocks that never fail and never delay → UI ships without loading/error states.
- Mock files left behind after the real endpoint exists.

## Checklist

- [ ] Contract types written first and match `api-design` conventions.
- [ ] Single mechanism (MSW or adapter); UI imports no mocks.
- [ ] Env flag controls mocks, defaults off, impossible in production.
- [ ] Mocks simulate latency, errors, pagination, and mutation effects.
- [ ] Each mock tracked with a pending-backend note.
- [ ] On integration: contract verified, mock removed or moved to tests.
