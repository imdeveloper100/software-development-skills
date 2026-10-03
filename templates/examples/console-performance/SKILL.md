---
name: console-performance
description: Performance rules specific to the admin console (the /console React app) — shell/layout data loading, route lazy-loading, the shared current-user/tenant/permissions queries, table pagination and lookup selectors. Use when adding or changing any page, layout, table, filter or data hook under apps/console, or when console navigation feels slow or the Network tab shows repeated requests.
---

# Console Performance (project-specific example)

> Example of a specialized, project-specific skill. It narrows the generic `react-performance`, `api-design` and `performance` skills to one app's concrete files, budgets and known problems. Copy into `.claude/skills/console-performance/` and replace paths/numbers with your project's.

## Purpose

Keep console navigation instant: each route change should trigger only the requests that route needs, and the shell should never re-download data the pages already have.

## Rules

### R1. Shell data is fetched once
`ConsoleLayout` may only use `useCurrentUser()`, `useActiveTenant()` and `usePermissions()` from `apps/console/src/features/auth/queries.ts`. These use `staleTime: 5 * 60_000` and keys from `authKeys`. Pages must reuse these hooks — never call `/auth/me` or `/tenants/current` directly.

### R2. Every route is lazy
All routes in `apps/console/src/routes.tsx` use the route `lazy` property, except `/console/dashboard`. Pages importing `recharts`, `@tiptap/*`, `xlsx` or `react-pdf` must load them via dynamic import inside the component that needs them.

### R3. Tables
- All console tables use `<DataTable>` + `usePaginatedQuery(keyFactory, fetcher, params)`; params live in URL search params via `useTableParams()`.
- Default `limit` 25, max 100; `placeholderData: keepPreviousData`.
- Never fetch a full list to compute counts — use the `/summary` endpoints.

### R4. Selectors
Every `<Select>`/`<Combobox>` bound to an entity uses its `/lookup` endpoint through `useLookup('<resource>', search)` (debounced 300 ms, `staleTime` 10 min). Adding a selector for a resource without a lookup endpoint → add the endpoint in `apps/api/src/modules/<resource>` first.

### R5. Budgets
- Initial console JS (gzipped) ≤ 250 KB — checked by `pnpm --filter console size` in CI.
- Max 4 API requests on first load of any page after the shell is loaded.
- Route change: 0 shell requests.

## Implementation Guidelines

1. Open DevTools Network → navigate dashboard → students → fees and back; count requests per navigation.
2. `rg "useQuery\(" apps/console/src` — any inline key not from a key factory is a finding.
3. Fix with shared hooks / key factories; never with `isLoaded` flags.
4. Run `pnpm --filter console size` and report the delta.

## Examples

```tsx
// ❌ page re-fetches the user the layout already has
const { data: me } = useQuery({ queryKey: ['me'], queryFn: api.me });

// ✅
const { data: me } = useCurrentUser();
```

## Anti-patterns

- `useEffect` fetching in console pages.
- Layout prefetching page-specific lists.
- `queryClient.invalidateQueries()` without a key after mutations.

## Checklist

- [ ] No new request in the shell; pages reuse auth hooks.
- [ ] New route lazy; heavy libs dynamically imported.
- [ ] Tables use DataTable + URL params; selectors use `useLookup`.
- [ ] Size budget check passes; request counts within budget.
