---
name: performance
description: Full-stack performance methodology — measure first, find the actual bottleneck, then apply the cheapest fix. Covers frontend (bundle size, code splitting, network requests, payload size, images, fonts, Core Web Vitals) and backend (query efficiency, N+1, indexes, pagination, caching and when Redis is justified, compression, connection pooling, async processing, serialization, response time). Use when something is slow, when asked to optimize, when adding caching/queues, or when reviewing performance-sensitive changes.
---

# Performance

## Purpose

Make the app measurably faster where users feel it, with the simplest change that fixes the real bottleneck — not speculative micro-optimizations. Details for React rendering/data-fetching live in `react-performance`; database specifics in `database`.

## Rules

### R1. Measure → locate → fix → re-measure
Never optimize without a measurement or a clear architectural concern (e.g., "this endpoint returns all rows of an unbounded table"). Record the before/after number in your report.

| Symptom | Measure with |
|---|---|
| Slow initial load | Lighthouse / PageSpeed, WebPageTest, bundle analyzer (`vite-bundle-visualizer`, `@next/bundle-analyzer`, `source-map-explorer`) |
| Janky interactions | React Profiler, Chrome Performance panel, INP in Web Vitals |
| Too many/slow requests | DevTools Network (waterfall, sizes, duplicates) |
| Slow endpoint | Server timing logs/APM (p50/p95), `Server-Timing` header, `autocannon`/`k6` |
| Slow query | `EXPLAIN ANALYZE`, Mongo `explain('executionStats')`, slow query log, ORM query logging |

### R2. Fix order (cheapest, highest-impact first)
1. **Do less work**: remove unneeded requests, fields, renders, dependencies.
2. **Fix the data access**: indexes, projections, pagination, N+1.
3. **Parallelize** independent I/O (`Promise.all`).
4. **Ship less**: code splitting, lazy loading, compression, image optimization.
5. **Cache** with explicit invalidation.
6. **Move work off the request path** (jobs/queues).
7. **Scale infrastructure** (bigger instance, replicas) — see `system-design`.

### R3. Readability budget
Don't trade clarity for unmeasurable gains. No manual loop unrolling, bit tricks, or caching of trivial computations. If an optimization makes code harder to read, it must come with a measured win and a comment explaining it.

## Frontend Rules

- **F1 Bundle**: initial JS for the main route should be justified; investigate any single dependency > ~50 KB gzipped in the initial bundle. Lazy-load routes and heavy libraries (charts, editors, maps, xlsx, pdf). Replace heavy libs with lighter equivalents when used trivially (moment → date-fns/dayjs or `Intl`).
- **F2 Network**: no duplicate requests; parallel not sequential; cache with the query library; prefetch next likely navigation only.
- **F3 Payload**: API responses for lists/lookups contain only displayed fields; paginate; gzip/brotli enabled. Flag responses > ~100 KB for a single screen.
- **F4 Rendering**: see `react-performance` C-rules; virtualize long lists.
- **F5 Images**: correct dimensions, WebP/AVIF, lazy below fold, explicit size to prevent CLS, `priority`/`fetchpriority="high"` for the LCP image only. Serve from CDN.
- **F6 Fonts**: self-host or `next/font`, subset, ≤ 2 families, limited weights, `font-display: swap`, preload the primary.
- **F7 Core Web Vitals targets** (public pages): LCP < 2.5 s, INP < 200 ms, CLS < 0.1.
- **F8 Static assets**: hashed filenames with `Cache-Control: public, max-age=31536000, immutable`; HTML with short/no cache.

## Backend Rules

- **B1 Queries**: projections, indexes justified by patterns, no N+1, pagination enforced (see `database`).
- **B2 Parallel I/O**: independent DB/HTTP calls in `Promise.all`; avoid `await` inside loops for independent items (use batched queries or bounded concurrency like `p-limit`).
- **B3 Payload & serialization**: response DTOs with needed fields only; avoid serializing huge object graphs; stream large exports (CSV) instead of building in memory.
- **B4 Compression**: enable gzip/brotli at the reverse proxy/CDN (preferred) or `compression()` middleware; skip already-compressed content.
- **B5 Connection pooling**: single pool per process, sized to DB limits; reuse HTTP agents (keep-alive) for outbound calls; set timeouts on all outbound calls.
- **B6 Caching decision** (from `backend-architecture` R12): fix query first → HTTP caching → in-process LRU → Redis when multi-instance sharing or high read volume demands it. Every cache entry needs TTL, invalidation trigger, tenant-safe key, and a stampede plan for hot keys (lock or stale-while-revalidate).
- **B7 Async processing**: move slow, non-user-blocking work (emails, PDFs, imports, webhooks, image processing) to jobs; return 202 for long operations.
- **B8 Event loop**: no CPU-heavy sync work on the request path (big JSON transforms, `bcrypt` sync, image processing, synchronous `fs`); use async APIs, worker threads, or jobs.
- **B9 Response time targets** (guide): p95 < 300 ms for typical CRUD reads, < 1 s for complex reports; anything slower is a candidate for pagination, pre-aggregation, or async.
- **B10 Observability**: log request duration; add per-endpoint timing; enable slow query logging in non-dev environments.

## Implementation Guidelines

1. Reproduce and measure (numbers + where).
2. Identify the dominant cost (network, JS, render, DB, external call).
3. Apply the fix from the earliest applicable step in R2.
4. Re-measure; report before/after.
5. Add a guard against regression where cheap (bundle size budget in CI, query count test, index in migration).

## Examples

**Sequential → parallel**
```ts
// ❌ ~3 round trips in series
const student = await getStudent(id);
const fees = await getFees(id);
const attendance = await getAttendance(id);
// ✅
const [student, fees, attendance] = await Promise.all([getStudent(id), getFees(id), getAttendance(id)]);
```

**Await in loop → batched**
```ts
// ❌ 500 queries
for (const row of rows) await prisma.student.update({ where: { id: row.id }, data: { status: row.status } });
// ✅ group by status, few queries (or a single transaction with bounded concurrency)
await prisma.$transaction(
  Object.entries(groupBy(rows, 'status')).map(([status, group]) =>
    prisma.student.updateMany({ where: { id: { in: group.map((r) => r.id) }, tenantId }, data: { status } })),
);
```

**Stream export instead of loading all rows**
```ts
res.setHeader('Content-Type', 'text/csv');
const cursor = Student.find({ tenantId }, { name: 1, email: 1 }).lean().cursor();
for await (const s of cursor) res.write(`${csvEscape(s.name)},${csvEscape(s.email)}\n`);
res.end();
```

## Anti-patterns

- Adding Redis before checking the query plan.
- `useMemo` everywhere while the page downloads 2 MB of JS.
- Returning 5,000 rows "because the table has client-side pagination".
- `await` inside `for` loops over independent I/O.
- Compression and caching done in Node when a CDN/proxy already does it (or neither does it).
- "Optimizations" without before/after numbers.
- Unreadable clever code for nanosecond gains.

## Checklist

- [ ] Bottleneck measured; numbers recorded before and after.
- [ ] Cheapest effective fix applied (less work → data access → parallel → ship less → cache → async → scale).
- [ ] Bundle: routes/heavy libs lazy; no oversized deps in initial load.
- [ ] Network: no duplicates/waterfalls; payloads slim; compression on.
- [ ] Backend: no N+1, projections, pagination, parallel I/O, pools, timeouts.
- [ ] Any cache has TTL, invalidation, tenant-safe key.
- [ ] Code remains readable; non-obvious optimizations commented.
