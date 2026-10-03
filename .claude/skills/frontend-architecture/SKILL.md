---
name: frontend-architecture
description: Structure rules for React and Next.js frontends — folder organization (feature vs shared), component boundaries, hooks, services/API clients, types, constants/static content, and choosing between server state, UI state, form state and global state. Use when creating or reorganizing components, pages, hooks, services, constants, or deciding where state should live.
---

# Frontend Architecture

## Purpose

Give every piece of frontend code an obvious home, keep components small and focused, and keep state in the right place — without creating a component, hook or file for every trivial thing.

## Rules

### R1. Follow the existing structure first
Inspect `src/` (or `app/`) before adding files. If the project is layer-based (`components/`, `hooks/`, `services/`) and small, keep it. Introduce feature folders only when a feature grows beyond ~5–6 files or the app has many distinct domains (admin panels, ERP modules).

### R2. Default structure for medium/large apps (feature-based)
```
src/
├── app/ or pages/ or routes/   # routing entry points only — thin
├── features/
│   └── students/
│       ├── components/          # used only by this feature
│       ├── hooks/               # useStudents, useStudentFilters
│       ├── api/                 # students.api.ts, students.queries.ts (query keys + hooks)
│       ├── types.ts
│       ├── constants.ts         # feature-local constants
│       └── index.ts             # public surface of the feature (optional)
├── components/
│   ├── ui/                      # design-system primitives (Button, Input, Modal, Table)
│   └── layout/                  # AppShell, Sidebar, Header, PageHeader
├── hooks/                       # truly generic hooks (useDebounce, useMediaQuery)
├── lib/                         # configured clients (apiClient, queryClient), framework glue
├── utils/                       # pure helpers split by topic (date.ts, currency.ts)
├── constants/                   # app-wide constants grouped by topic
├── config/                      # runtime/app config (env access, feature toggles)
├── content/                     # static marketing/page copy
└── types/                       # cross-feature/shared types
```
- A feature may import from `components/ui`, `hooks`, `lib`, `utils`, `constants`. Features should **not** deep-import another feature's internals; promote shared code to a shared folder or import via the feature's `index.ts`.
- Route files (`page.tsx`, route components) compose features; they hold minimal logic.

### R3. Component boundaries
Create a separate component when at least one is true:
- It is reused (or clearly will be in this task).
- It has its own state/effects/data that would otherwise re-render a large parent.
- It represents a meaningful UI concept (`StudentRow`, `InvoiceSummary`, `FilterBar`).
- The parent exceeds ~200–250 lines of JSX+logic and splitting improves readability.

Do **not** create components for: a single styled `<div>`, one-off wrappers that just pass props through, or "Title"/"Text" wrappers that duplicate what the design system or Tailwind classes already give.

Component types:
- **UI primitives** (`components/ui`): no data fetching, no business logic, fully prop-driven, use design tokens.
- **Feature components**: may use feature hooks/queries.
- **Page/route components**: composition + route params → feature components.
- **Layout components**: shell, nav, sidebars. Keep them lightweight; don't fetch page-specific data in layouts.

### R4. Hooks
- Extract a custom hook when logic (state + effects + derived values) is reused or when it makes the component read like a description of UI.
- Name by purpose: `useStudentFilters`, `useEnrollStudent` — not `useData`, `useHelper`.
- Data hooks wrap the query library (`useStudentsQuery`) — components should not build query keys inline in many places.
- Don't wrap a single `useState` in a hook.

### R5. Services / API layer
- One configured HTTP client in `lib/api-client.ts` (base URL from env, auth header/credentials, error normalization, interceptors).
- Per-resource API modules: `features/students/api/students.api.ts` exporting typed functions (`getStudents(params)`, `createStudent(dto)`).
- Components never call `axios`/`fetch` directly with hardcoded URLs.
- Query hooks + query key factories live next to the API functions:
```ts
// students.queries.ts
export const studentKeys = {
  all: ['students'] as const,
  list: (params: StudentListParams) => [...studentKeys.all, 'list', params] as const,
  detail: (id: string) => [...studentKeys.all, 'detail', id] as const,
  lookup: (search: string) => [...studentKeys.all, 'lookup', search] as const,
};
```
- If the backend endpoint doesn't exist yet → follow `mock-first-development`.

### R6. State placement (decide in this order)
| Kind of state | Where it lives |
|---|---|
| Data from the server (lists, entities, lookups) | **Server-state cache**: TanStack Query / SWR / RTK Query (whatever the project uses). Never copy it into `useState`/Redux/context. |
| Derivable from props/other state | **Compute during render** (no state, no effect). |
| UI state for one component (open, active tab, hover) | `useState` in that component. |
| UI state shared by a few nearby components | Lift to the nearest common parent. |
| URL-worthy state (filters, page, sort, selected tab that should survive refresh/share) | **URL search params**. |
| Form state | Form library (React Hook Form + zod resolver) or local state for tiny forms. |
| Truly global client state (theme, auth session snapshot, sidebar collapsed, multi-step wizard spanning routes) | Context (low-frequency updates) or Zustand/Redux **if already in the project or clearly needed**. |

- Do not introduce Redux/Zustand for state that one of the first five rows covers.
- Context is for low-frequency values. High-frequency values in context re-render every consumer — split contexts or use a store with selectors.

### R7. Constants and static content
- Small values used in one component stay local.
- Values used across files or long lists (nav items, statuses, permissions, pricing plans, countries, select options, page copy) move out of components:
```
constants/
├── navigation.ts      # NAV_ITEMS (route, label, icon, requiredPermission)
├── permissions.ts     # PERMISSIONS, role → permission maps (mirrors backend for UX only)
├── statuses.ts        # STUDENT_STATUS + label/color-token maps
└── countries.ts
content/
├── homepage.ts        # hero copy, feature blocks
└── pricing.ts         # plans, feature matrix
config/
└── app.ts             # app name, support email, pagination defaults
```
- **No** single giant `constants.ts`. Group by topic.
- Use `as const` and derive types from constants:
```ts
export const STUDENT_STATUS = ['active', 'suspended', 'graduated'] as const;
export type StudentStatus = (typeof STUDENT_STATUS)[number];
```
- Labels and display maps sit next to the enum they describe.
- Dynamic data (anything an admin can edit) belongs in the backend, not in constants.

### R8. Types
- Feature types in the feature (`types.ts`); cross-feature types in `types/`.
- API DTO types mirror backend contracts (ideally generated from OpenAPI or a shared package). Don't hand-maintain divergent copies.
- Distinguish API shapes from view models when they differ (`StudentDto` vs `StudentRowViewModel`); map in the API/query layer (`select`), not in JSX.

### R9. Every data-driven view handles all states
Loading (skeleton matching layout), error (message + retry), empty (helpful copy + primary action), success. Plus 401 (redirect to login), 403 (forbidden message), 404 (not-found UI), validation errors (inline per field from the server's `details`).

### R10. Styling
Use the project's styling system and tokens — see `design-system`. No hardcoded hex colors or arbitrary spacing in components.

## Implementation Guidelines

1. Identify the feature/domain. Does a folder already exist? Add to it.
2. Check `components/ui` and the feature for something reusable before creating a component.
3. Add API function → query hook (with key factory) → component using the hook.
4. Put long static arrays/config in `constants/`, `content/` or `config/`.
5. Make the route/page file a thin composition layer.
6. Handle loading/error/empty states.

## Examples

**Thin page, feature components, hook-driven data**
```tsx
// app/(dashboard)/students/page.tsx  OR  routes/StudentsPage.tsx
export default function StudentsPage() {
  return (
    <>
      <PageHeader title="Students" actions={<CreateStudentButton />} />
      <StudentsTable />
    </>
  );
}

// features/students/components/StudentsTable.tsx
export function StudentsTable() {
  const [params, setParams] = useStudentListParams();       // synced to URL search params
  const { data, isPending, isError, refetch } = useStudentsQuery(params);

  if (isPending) return <TableSkeleton rows={10} />;
  if (isError) return <ErrorState onRetry={refetch} />;
  if (data.items.length === 0) return <EmptyState title="No students yet" action={<CreateStudentButton />} />;

  return <DataTable rows={data.items} pagination={data.meta} onParamsChange={setParams} />;
}
```

**Derived state, not effect + state**
```tsx
// ❌
const [fullName, setFullName] = useState('');
useEffect(() => setFullName(`${first} ${last}`), [first, last]);

// ✅
const fullName = `${first} ${last}`;
```

**Navigation config instead of JSX lists**
```ts
// constants/navigation.ts
export const NAV_ITEMS = [
  { label: 'Dashboard', href: '/dashboard', icon: LayoutDashboard },
  { label: 'Students', href: '/students', icon: Users, permission: 'students.read' },
  { label: 'Fees', href: '/fees', icon: Wallet, permission: 'fees.read' },
] as const;
```

## Anti-patterns

- 800-line page components mixing fetching, tables, modals and forms.
- Copying query data into `useState` or a global store (`setStudents(data)`), creating two sources of truth.
- Components calling `axios.get('http://localhost:4000/...')` directly.
- A `Wrapper`/`Box`/`Text` component for every HTML element without added value.
- Global context provider for state used by one page.
- Deep imports across features (`features/fees/components/internal/FeeRow`).
- One `constants.ts` with 600 lines of unrelated data.
- Filters/pagination in local state when users expect them to survive refresh or be shareable.
- Business-critical rules (pricing, permission checks) implemented only in the UI.

## Checklist

- [ ] Files placed according to existing structure (or feature folders, if justified).
- [ ] Page/route files are thin; features hold the logic.
- [ ] No component created without reuse, isolation, or clarity benefit.
- [ ] Server data lives only in the query cache; derived values computed, not stored.
- [ ] Filters/sort/page in URL where users expect persistence.
- [ ] API calls go through the shared client + per-resource API module.
- [ ] Static lists/content moved to `constants/`, `content/` or `config/`, grouped by topic.
- [ ] Loading, error, empty, 401/403/404 and validation states handled.
- [ ] Design tokens used; no hardcoded colors.
