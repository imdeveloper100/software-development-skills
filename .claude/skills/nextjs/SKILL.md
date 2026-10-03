---
name: nextjs
description: Next.js rules (App Router first, Pages Router aware) — Server vs Client Components, server-side data fetching, caching and revalidation, static vs dynamic rendering, streaming with loading.tsx/Suspense, error boundaries, Server Actions vs Route Handlers, middleware/proxy, metadata/SEO (sitemap, robots, Open Graph, JSON-LD), next/image, next/font, dynamic imports and env vars. Use for any work in a Next.js project.
---

# Next.js

## Purpose

Use Next.js the way it is designed: render on the server by default, ship minimal client JavaScript, cache deliberately, and make pages fast and indexable.

## Rules

### R0. Check the version and router first
Read `next` in `package.json` and check for `app/` vs `pages/`. Caching defaults and APIs differ between versions (e.g., `fetch` is **not** cached by default since Next 15; newer versions add `"use cache"`/Cache Components; Next 16 renames `middleware.ts` to `proxy.ts`). Follow the APIs of the installed version and the pattern already used in the repo. Don't migrate Pages → App Router as a side effect of a feature.

### R1. Server Components by default
- Every component in `app/` is a Server Component unless it needs: state/effects, event handlers, browser APIs, or client-only libraries.
- Put `'use client'` on the **smallest leaf** that needs interactivity, not on pages or layouts.
- Pattern: Server page fetches data → passes serializable props to a small Client component.
- Server Components can render Client Components; Client Components receive Server Components only via `children`/props.
- Never import server-only modules (DB clients, secrets) into client files. Mark server modules with `import 'server-only'`.

### R2. Fetch data on the server when you can
- Data needed for the initial render (page content, SEO-relevant data, user-specific dashboard data) → fetch in the Server Component, directly from the service/DB layer or backend API.
- Do **not** create a Route Handler just so your own Server Component can `fetch` it — call the function directly.
- Run independent fetches in parallel: `const [a, b] = await Promise.all([getA(), getB()])`.
- Deduplicate per request with `React.cache()` for non-`fetch` data functions used in multiple components (e.g., `getCurrentUser`).
- Use client-side fetching (TanStack Query/SWR) for: highly interactive data (live search, infinite scroll, polling), data that changes after user interaction without navigation, or when the backend is a separate API consumed with user tokens in the browser.
- Hybrid is fine: prefetch on server and hydrate the query cache (`HydrationBoundary`) when a page is both SEO-relevant and highly interactive.

### R3. Caching & revalidation — be explicit
Decide per data source and write it in code:
| Data | Strategy |
|---|---|
| Marketing/blog/docs content | Static + time revalidation (`export const revalidate = 3600`) or on-demand `revalidateTag`/`revalidatePath` from CMS webhook |
| Product catalog / public lists | Cached with tags; revalidate on mutation |
| User-specific / auth-dependent | Dynamic (`cookies()`/`headers()` usage makes it dynamic); no shared cache |
| Real-time | Client-side polling/websocket, not server cache |

- After a mutation (Server Action / Route Handler), call `revalidateTag('students')` or `revalidatePath('/students')` for affected data.
- Never cache user-specific responses in a shared cache.
- Avoid `export const dynamic = 'force-dynamic'` as a blanket fix — use it only when the page truly must be dynamic and say why.

### R4. Rendering modes
- Static where content is the same for all users (marketing, docs, blog). Use `generateStaticParams` for known dynamic segments.
- Dynamic where content depends on request (auth, cookies, search params).
- Keep static shells static: isolate dynamic parts behind `Suspense` so the rest can stream/prerender.

### R5. Loading, streaming, errors
- Add `loading.tsx` for route segments with meaningful data latency; use skeletons matching the layout.
- Wrap slow, independent sections in `<Suspense fallback={...}>` so fast content renders first.
- Add `error.tsx` (Client Component with `reset`) for segments that can fail; `global-error.tsx` at root; `not-found.tsx` and call `notFound()` for missing entities.
- Don't throw raw backend error text into the UI.

### R6. Mutations: Server Actions vs Route Handlers
- **Server Actions**: form submissions and mutations from this Next app's own UI. Validate input with zod **inside** the action; check auth/authorization **inside** the action (actions are public endpoints). Return typed results `{ ok: true } | { ok: false, fieldErrors }`; revalidate afterwards.
- **Route Handlers** (`app/api/.../route.ts`): webhooks, endpoints for external clients/mobile apps, file streams, OAuth callbacks, or when a separate backend isn't present and you need a REST API.
- If the project has a separate backend (NestJS/Express), Next is a client of it — don't duplicate business logic in Route Handlers. A thin BFF proxy is acceptable when it hides tokens or aggregates calls.

### R7. Middleware / proxy
- Use for cheap, request-level concerns: auth redirect based on session cookie presence, locale redirect, rewrites, headers.
- Do **not** do DB queries or heavy logic there; it runs on every matched request. Use a precise `matcher` that excludes static assets (`_next/static`, images, favicon).
- Middleware redirect is UX only — authorization must still be enforced in Server Components/Actions/Route Handlers and the backend.

### R8. Metadata & SEO (public pages)
- Static: `export const metadata: Metadata = {...}`. Dynamic: `generateMetadata({ params })` reusing the same cached data function as the page (no double fetch).
- Set `metadataBase`, `title.template` in root layout, canonical (`alternates.canonical`), `openGraph`, `twitter`.
- `app/sitemap.ts` and `app/robots.ts` generated from real routes/data; exclude auth/admin/dashboard routes and noindex them.
- JSON-LD via a `<script type="application/ld+json">` in the Server Component with `JSON.stringify(data).replace(/</g, '\\u003c')`.
- Dashboard/admin pages: `robots: { index: false }`; don't spend effort on SEO there.

### R9. Images and fonts
- `next/image` for content images with `width/height` or `fill` + `sizes`; `priority` only for the LCP image (usually one per page).
- Configure `images.remotePatterns` for external hosts — narrowly.
- `next/font` (Google or local) in the root layout, exposed as a CSS variable used by the design system; limit weights/subsets.

### R10. Code splitting
- `next/dynamic(() => import('./Chart'), { ssr: false, loading: () => <ChartSkeleton/> })` for heavy client-only widgets (charts, editors, maps).
- Don't import heavy client libraries into layouts.
- Check bundle impact with `@next/bundle-analyzer` when adding large deps.

### R11. Environment variables
- Server-only secrets: no prefix, read only in server code.
- `NEXT_PUBLIC_*` is inlined into the client bundle at **build time** — public values only, and changing them needs a rebuild (matters for Docker images promoted across environments; see `docker`).
- Validate env with a schema (e.g., `@t3-oss/env-nextjs` or zod) if the project does.

### R12. Auth in Next
- Read the session on the server (`auth()`/cookie session) in layouts/pages that need it; pass only needed user fields to Client Components.
- Protect: middleware for redirect UX + server-side checks in every page/action/handler that reads protected data.
- Use httpOnly cookies for tokens; never expose refresh tokens to client JS. See `authentication`.

## Implementation Guidelines

1. Identify version + router + data source (direct DB, internal services, or external API).
2. Build the page as a Server Component that fetches in parallel.
3. Extract interactive bits into small `'use client'` components.
4. Choose and code the caching/revalidation strategy; add tags.
5. Add `loading.tsx` / Suspense boundaries and `error.tsx` / `notFound()`.
6. For public pages, add metadata, canonical, OG; update sitemap if new routes.
7. Run `next build` and inspect the route table (static ○ vs dynamic ƒ) to confirm the rendering mode you intended.

## Examples

**Server page + small client island**
```tsx
// app/(dashboard)/students/[id]/page.tsx  (Server Component)
import { notFound } from 'next/navigation';
import { getStudent, getStudentFees } from '@/features/students/server/queries';
import { EnrollButton } from '@/features/students/components/EnrollButton';

export default async function StudentPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const [student, fees] = await Promise.all([getStudent(id), getStudentFees(id)]);
  if (!student) notFound();

  return (
    <>
      <StudentHeader student={student} />
      <FeesSummary fees={fees} />
      <EnrollButton studentId={student.id} />  {/* 'use client' leaf */}
    </>
  );
}
```

**Server Action with validation, authz, revalidation**
```ts
'use server';
import { z } from 'zod';
import { revalidateTag } from 'next/cache';

const Input = z.object({ name: z.string().min(2).max(100), classId: z.string().uuid() });

export async function createStudent(_: unknown, formData: FormData) {
  const session = await requireSession();                    // throws/redirects if absent
  assertPermission(session, 'students.create');               // server-side authz
  const parsed = Input.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors };

  await studentService.create(session.tenantId, parsed.data);
  revalidateTag('students');
  return { ok: true };
}
```

**Per-request dedupe**
```ts
import { cache } from 'react';
export const getCurrentUser = cache(async () => { /* read session, load user once per request */ });
```

## Anti-patterns

- `'use client'` at the top of `page.tsx` or `layout.tsx` "to make it work".
- `useEffect` + `fetch('/api/...')` in a Client Component for data the server could render.
- Route Handlers that only exist to be fetched by the app's own Server Components.
- `force-dynamic` everywhere; or caching user-specific data in shared cache.
- Server Actions without validation or authorization checks.
- DB calls in middleware.
- Secrets in `NEXT_PUBLIC_*`.
- `<img>` for content images; `priority` on many images.
- Fetching the same data separately in `generateMetadata` and the page without dedupe.

## Checklist

- [ ] Version/router checked; APIs used match the installed version.
- [ ] Server Components by default; `'use client'` only on interactive leaves.
- [ ] Initial data fetched on the server, in parallel, deduped.
- [ ] Caching/revalidation strategy explicit; mutations revalidate affected tags/paths.
- [ ] `loading`/Suspense, `error`, `not-found` handled.
- [ ] Server Actions/Route Handlers validate input and enforce authz.
- [ ] Public pages: metadata, canonical, OG, sitemap/robots; private pages noindex.
- [ ] `next/image`, `next/font`, heavy widgets via `next/dynamic`.
- [ ] No secrets in client bundle; `next build` passes with intended static/dynamic modes.
