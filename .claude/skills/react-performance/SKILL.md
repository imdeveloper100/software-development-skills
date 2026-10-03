---
name: react-performance
description: Client-side React performance and data-fetching rules — TanStack Query/SWR over useEffect fetching, query keys and caching, duplicate/waterfall request detection, lazy-loading routes and heavy libraries, re-render control, memoization judgment, virtualization, pagination/infinite scroll, debounce/throttle. Use when building or reviewing React/Next.js client components that fetch data, render lists, or feel slow, and when auditing for duplicate API calls.
---

# React Performance

## Purpose

Ship less JavaScript, make fewer and smaller requests, and render less — using measured, readable techniques instead of reflexive `useMemo` everywhere.

## Rules

### A. Data fetching (server state)

**A1. Use the query library for server state.** If TanStack Query / SWR / RTK Query is present, all server reads go through it. If none is present and the app fetches server data in more than a couple of places, recommend adding TanStack Query (explain why) rather than hand-rolling caching.

```tsx
// ❌ manual server-state fetching
const [data, setData] = useState();
useEffect(() => { api.getStudents().then(setData); }, []);

// ✅
const { data, isPending, error } = useQuery({
  queryKey: studentKeys.list(params),
  queryFn: () => getStudents(params),
  staleTime: 60_000,
});
```

`useEffect` remains correct for: DOM measurement/focus, subscriptions (WebSocket, event listeners), timers, browser APIs (`IntersectionObserver`, `localStorage` sync), and syncing with non-React widgets. Always return a cleanup.

**A2. Query keys** are built by a key factory per resource (see `frontend-architecture` R5). Keys include **every** input of the `queryFn` (filters, page, id, tenant if it can change). Never duplicate the same data under two different key shapes.

**A3. Configure caching deliberately.** Set a `QueryClient` default and override per query:
| Data | Suggested `staleTime` |
|---|---|
| Static/reference lookups (countries, departments, roles) | 10–60 min or `Infinity` with explicit invalidation |
| Lists users edit often | 0–30 s |
| Current user / session | 5 min, invalidate on login/logout |
| Dashboards/analytics | 1–5 min |

Sensible defaults to consider: `refetchOnWindowFocus: false` for admin/ERP apps where focus refetch is noise (keep it on where freshness matters), `retry: 1–2` for GETs, no retry on 4xx.

**A4. No duplicate requests.**
- The same resource requested by layout and page → fetch once via the shared query hook; both read from cache.
- Multiple components needing the same data → call the same hook with the same key; the library dedupes. Don't add `isFetched` flags or prop-drill to "avoid" a request.
- After mutations, update via `invalidateQueries` on the **narrowest** key, or `setQueryData` when the response contains the updated entity — don't refetch the world.
- Don't refetch on route change unless the data depends on the route param.

**A5. No waterfalls.** Independent requests run in parallel:
```tsx
// ❌ sequential: second waits for first though it doesn't depend on it
const { data: user } = useQuery(userQuery);
const { data: stats } = useQuery({ ...statsQuery, enabled: !!user });

// ✅ parallel (both start immediately)
const [user, stats] = useQueries({ queries: [userQuery, statsQuery] });
```
Use `enabled` only for **true** dependencies (need `user.orgId` to fetch org data). Better still, ask whether the backend should provide one purpose-built endpoint.

**A6. Fetch only what the UI uses.** Dropdowns/autocomplete use `/resource/lookup` (id + label). Tables request paginated, projected lists. If the endpoint over-returns, propose a slimmer endpoint or `fields` param (see `api-design`). Use `select` to shape data for the component, not to hide over-fetching.

**A7. Large datasets** → server pagination or infinite query (`useInfiniteQuery`) — never fetch all rows and paginate on the client unless the dataset is small and bounded (< ~500 rows).

**A8. Search inputs** are debounced (250–400 ms) before they become part of a query key; use `placeholderData: keepPreviousData` to avoid flashing empty states.

**A9. Prefetch where it's free:** on hover/focus of a link to a detail page, or the next page of a paginated list. Don't prefetch everything at startup.

### B. Code loading

**B1. Route-level code splitting.** In SPA routers, lazy-load route components (except the landing/shell route):
```tsx
const AdmissionSettingsPage = lazy(() => import('./pages/AdmissionSettings'));
// React Router: <Suspense fallback={<PageSkeleton />}> around <Outlet /> or the route element
// or use the router's `lazy` route property
```
Next.js App Router splits per route automatically — use `next/dynamic` for heavy client components instead (see `nextjs`).

**B2. Lazy-load heavy components and libraries** used conditionally: charts, rich-text editors, maps, PDF viewers, date-heavy pickers, code editors, Excel export (`xlsx`), large modals/wizards.
```tsx
const RevenueChart = lazy(() => import('./RevenueChart'));
async function exportToExcel(rows) {
  const XLSX = await import('xlsx');       // loaded only when the user clicks Export
  ...
}
```

**B3. Keep the shell light.** Layout/sidebar/header must not import page-specific libraries or fetch page data.

**B4. Import precisely.** `import debounce from 'lodash/debounce'` (or `lodash-es`) not `import _ from 'lodash'`; icon libraries via named imports that tree-shake; avoid barrel files that pull whole modules into the shell when the bundler can't tree-shake them.

### C. Rendering

**C1. Measure before optimizing.** Use React DevTools Profiler ("Highlight updates") to find components that re-render excessively or are slow (> ~16 ms). Optimize those.

**C2. Fix structure before memoizing:**
- Move state down to the component that uses it.
- Pass children as props (`<Layout>{children}</Layout>`) so state changes in the wrapper don't re-render the children.
- Split a context whose value changes often from one that rarely changes; memoize the provider `value`.
- Don't put fast-changing state (input text, mouse position) in global context.

**C3. Memoization — use when there is a real benefit:**
- `React.memo`: a component that renders often with the same props and is expensive (table rows, chart, list items in large lists).
- `useMemo`: expensive computations (sorting/filtering thousands of items, heavy transforms) or to keep a referentially stable object/array that is a dependency of a memoized child or an effect.
- `useCallback`: when the function is passed to a `memo` child or used in a dependency array.
- **Do not** memoize cheap expressions (`a + b`, string concat, small maps) or callbacks passed to plain DOM elements.
- If the project uses the React Compiler, skip manual memoization unless the profiler shows a problem.

**C4. Lists**
- Stable, unique `key` from data (`id`), never array index for lists that reorder/insert/delete.
- Virtualize when rendering more than ~100–200 complex rows or ~1000 simple ones (`@tanstack/react-virtual`, `react-window`) — or paginate.
- Don't define components inside other components (remounts every render).

**C5. High-frequency events:** throttle scroll/resize/mousemove handlers (or use `requestAnimationFrame` / `IntersectionObserver`); debounce text-driven requests and autosave.

**C6. Forms:** prefer uncontrolled forms with React Hook Form for large forms; avoid a whole-form re-render per keystroke.

**C7. Use `useTransition`/`useDeferredValue`** for expensive client-side filtering so typing stays responsive.

### D. Assets
Images: correct size, modern formats, `loading="lazy"` below the fold, explicit width/height to avoid layout shift (in Next.js use `next/image`). Fonts: self-hosted/`next/font`, `font-display: swap`, limited weights.

## Implementation Guidelines — duplicate-request audit

When asked to find/fix duplicate or unnecessary requests:
1. Open the app with DevTools Network (Fetch/XHR) and navigate the affected flow; note repeated URLs.
2. In code, `rg "useQuery|useSWR|axios\.|fetch\(" src` and group by endpoint.
3. Look for: same endpoint under different query keys; layout + page fetching the same resource; `useEffect` fetches that run on every mount; `enabled` chains that serialize independent requests; `invalidateQueries()` with no key; `refetchOnMount: 'always'`/`staleTime: 0` on reference data; React StrictMode double-invoked effects (dev only — fixed by using the query library, not by disabling StrictMode).
4. Fix by consolidating into one query hook + key factory and tuning `staleTime`, not by adding booleans/flags.
5. Report findings as: endpoint → where called → why duplicated → fix.

## Examples

**Debounced lookup with stable cache**
```tsx
function useDepartmentLookup(search: string) {
  const debounced = useDebounce(search.trim(), 300);
  return useQuery({
    queryKey: departmentKeys.lookup(debounced),
    queryFn: () => getDepartmentLookup({ search: debounced, limit: 20 }),
    staleTime: 5 * 60_000,
    placeholderData: keepPreviousData,
  });
}
```

**Optimistic-free precise invalidation**
```tsx
const updateStudent = useMutation({
  mutationFn: patchStudent,
  onSuccess: (student) => {
    queryClient.setQueryData(studentKeys.detail(student.id), student);
    queryClient.invalidateQueries({ queryKey: studentKeys.all, refetchType: 'active' }); // lists only if mounted
  },
});
```

## Anti-patterns

- `useEffect(() => { fetch(...).then(setX) }, [])` for server data when a query library exists.
- Storing query results in Redux/Zustand/context.
- `queryClient.invalidateQueries()` with no key after every mutation.
- Fetching `/users` (full objects) to fill a dropdown.
- Fetching 10,000 rows and paginating in the browser.
- Every function wrapped in `useCallback`, every value in `useMemo`.
- Index as `key` on editable/sortable lists.
- Statically importing every page into `App.tsx`.
- Importing a chart library in the root layout.
- Disabling StrictMode to "fix" double requests.

## Checklist

- [ ] Server state via the query library with key factories; no `useEffect` fetching.
- [ ] `staleTime` chosen per data type; reference data cached long.
- [ ] No duplicate requests across layout/page/siblings; mutations invalidate narrowly.
- [ ] Independent requests run in parallel; `enabled` only for real dependencies.
- [ ] Dropdowns use lookup endpoints; lists paginated; search debounced.
- [ ] Routes and heavy libraries lazy-loaded; shell imports are light.
- [ ] Memoization justified by profiler or clear cost; none on trivial values.
- [ ] Large lists virtualized or paginated; stable keys.
