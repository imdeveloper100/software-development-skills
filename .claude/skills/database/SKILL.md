---
name: database
description: Database rules for MongoDB (Mongoose) and PostgreSQL (Prisma/TypeORM/Drizzle/Knex) — inspecting existing schemas, schema design, embedding vs referencing, normalization, constraints, justified indexes, projections/selects, avoiding N+1 and over-fetching, populate/joins, aggregation, pagination queries, transactions, safe migrations, connection pooling and query analysis with EXPLAIN. Use when touching schemas, models, migrations, queries, or investigating slow database access.
---

# Database

## Purpose

Correct data first (constraints, transactions, safe migrations), then efficient access (right indexes, minimal data fetched, no N+1) — always based on actual query patterns.

## Rules — both databases

### R1. Inspect before changing
Read the existing schema/models, migrations history, indexes (`\d table` / `db.collection.getIndexes()` / ORM schema), and how the data is queried today. Don't add a column/field that already exists under another name. Follow existing naming (snake_case columns vs camelCase).

### R2. Fetch only what you need
- Always select/project fields for list and lookup queries. No `SELECT *` / bare `find()` in API paths.
- Never load collections to count them (`(await find()).length`) — use `count`.
- Never filter/paginate in application memory what the DB can do.

### R3. No N+1
A query inside a loop over results is N+1. Fix with: `include`/join, `populate` (with `select`), `$lookup`, or a single `WHERE id IN (...)` followed by a map. Watch for N+1 hidden in serializers/resolvers that lazily load relations.

### R4. Index justification (never add blindly)
Add an index only for a concrete query pattern, and state it in the PR/migration comment:
> "Index `(tenant_id, status, created_at DESC)` supports `GET /students?status=…` sorted by newest — the main list screen. Without it the query seq-scans ~500k rows."

Rules of thumb:
- Equality fields first, then sort field, then range fields (ESR rule — also applies to Postgres composite B-tree).
- Multi-tenant tables: `tenant_id` leads most composite indexes.
- Unique business keys → unique index/constraint (`(tenant_id, email)`).
- Every foreign key used in joins/filters is indexed (Postgres doesn't auto-index FKs).
- Each index costs write performance and storage; drop unused/duplicate ones (a prefix of another composite index is usually redundant).
- Verify with `EXPLAIN (ANALYZE, BUFFERS)` (Postgres) or `.explain('executionStats')` (Mongo): look for Seq Scan / COLLSCAN on large tables, and `totalDocsExamined` ≫ `nReturned`.

### R5. Pagination queries
- Offset: `LIMIT/OFFSET` / `skip/limit` — fine for moderate offsets; degrades with deep pages.
- Keyset/cursor: `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20` — use for large tables/infinite scroll. Requires matching index.
- Always deterministic `ORDER BY` incl. unique tie-breaker.

### R6. Transactions
Use for multi-record invariants (see `backend-architecture` R10). Keep them short, no network calls inside, handle retry on serialization/write conflicts where the DB may raise them.

### R7. Migrations & existing data
- Schema changes go through migrations (Prisma Migrate, TypeORM migrations, Knex, or a Mongo migration tool like `migrate-mongo` for data reshaping). Never `synchronize: true` / `db push` against shared/prod DBs.
- Zero-downtime pattern for breaking changes — **expand → migrate → contract**: add new nullable column/field → backfill (batched) → deploy code reading new → make NOT NULL / drop old in a later release.
- Adding NOT NULL to an existing table: add nullable + default/backfill first.
- Large table indexes in Postgres: `CREATE INDEX CONCURRENTLY` (outside a transaction).
- Renames are breaking: add + copy + switch + drop.
- Destructive migrations (drop column/table) require explicit confirmation and a backup note.

### R8. Connections
- One client/pool per process (singleton). Never create a client per request.
- Size pools to DB limits: `instances × pool size < max_connections` (leave headroom). Serverless → use a pooler (PgBouncer, RDS Proxy, Prisma Accelerate/Data Proxy, Neon pooler).
- Set statement/query timeouts for API paths.

### R9. Soft delete & audit columns (when the domain needs them)
`createdAt`, `updatedAt` everywhere; `deletedAt` only when restore/history is a requirement — and then every query must filter it (use a default scope/middleware) and unique indexes must account for it (partial index `WHERE deleted_at IS NULL`).

## Rules — PostgreSQL

### P1. Schema design
- Normalize to 3NF by default; denormalize deliberately (counts, snapshots on invoices) with a documented sync strategy.
- Constraints enforce invariants: `NOT NULL`, `UNIQUE`, `CHECK (amount_cents >= 0)`, `FOREIGN KEY` with explicit `ON DELETE` behavior (`RESTRICT` default; `CASCADE` only for true ownership).
- Types: `uuid` or `bigint identity` PKs (follow project), `timestamptz` for times, `numeric(12,2)` or integer cents for money, `text` over arbitrary `varchar(n)` unless a limit is a business rule, `jsonb` only for truly schemaless attributes (index with GIN if queried).
- Enums: Postgres enum or lookup table; lookup table if values change often or need metadata.

### P2. Querying
- Use ORM `select`/`include` deliberately; check generated SQL for heavy includes.
- Search: `ILIKE 'abc%'` can use a `text_pattern_ops`/`lower()` index; `%abc%` needs `pg_trgm` GIN index; full-text needs `tsvector` + GIN.
- Aggregations in SQL (`GROUP BY`, `COUNT`, `SUM`), not in JS.
- Raw SQL must be parameterized (`$1`, Prisma tagged template `$queryRaw\`...${x}\``) — never string-concatenated. Avoid `$queryRawUnsafe` with user input.

## Rules — MongoDB

### M1. Schema design
- Design around access patterns. **Embed** when data is owned by the parent, read together, and bounded (address, a few settings, line items of an invoice). **Reference** when unbounded (students of a class, logs, comments), shared across parents, or updated independently.
- Never let an array grow unbounded inside a document (16 MB limit + slow updates).
- Use Mongoose schema validation (`required`, `enum`, `min`, `match`), `timestamps: true`, and `strict` mode. Add `tenantId` to every tenant-owned collection.
- Denormalize small, rarely-changing fields for read performance (e.g., `className` on a student) only with an update strategy.

### M2. Querying
- `.select('name code')` / projection on every list/lookup; `.lean()` for read-only queries (plain objects, much faster, no getters/virtuals — confirm you don't need them).
- `populate` only with `select` and only for fields displayed; never populate arrays that can be large — paginate the child collection instead. For complex joins/aggregates prefer one `$lookup` pipeline.
- Aggregation: `$match` (indexed) first, then `$project` early to shrink documents, then `$group`/`$lookup`; use `$facet` for items + total in one round trip when needed.
- Escape user input in regex: `new RegExp(escapeRegex(search), 'i')`; anchored prefix regex (`^abc`) can use an index; case-insensitive search at scale → collation index or a normalized lowercase field, or Atlas Search.
- Prevent operator injection: cast/validate types (`String(input)`), use `mongo-sanitize`/`sanitizeFilter`, never `find(req.body)`.
- Atomic updates: `$set`, `$inc`, `$push` with conditions in filter (`{ _id, seatsLeft: { $gt: 0 } }`) instead of read-modify-write.

### M3. Transactions
Multi-document transactions need a replica set (Atlas has one; local dev may need `--replSet`). Prefer single-document atomicity via modeling where possible; use `session.withTransaction()` when multiple documents must change together.

## Implementation Guidelines

1. Read current schema + migrations + indexes; find all queries touching the entity (`rg "student\.(find|findMany|aggregate)"`).
2. Write the query with selects/projections and correct pagination.
3. Check the query plan for important queries; add a justified index via migration if needed.
4. For schema changes, write an expand/contract migration and a backfill plan.
5. Test against realistic data volume if performance matters (seed script).

## Examples

**N+1 → single query**
```ts
// ❌ N+1
const invoices = await prisma.invoice.findMany({ where: { tenantId } });
for (const inv of invoices) inv.student = await prisma.student.findUnique({ where: { id: inv.studentId } });

// ✅
const invoices = await prisma.invoice.findMany({
  where: { tenantId }, take: 20, orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
  select: { id: true, amountCents: true, status: true, student: { select: { id: true, name: true } } },
});
```

**Mongo lookup query**
```ts
const items = await Student.find(
  { tenantId, status: 'active', name: { $regex: `^${escapeRegex(search)}`, $options: 'i' } },
  { name: 1 },
).sort({ name: 1 }).limit(20).lean();
// Supporting index: { tenantId: 1, status: 1, name: 1 } — equality, equality, sort/prefix.
```

**Safe column addition (Postgres)**
```sql
-- 1) expand
ALTER TABLE students ADD COLUMN admission_no text;
-- 2) backfill in batches (script), then
CREATE UNIQUE INDEX CONCURRENTLY students_tenant_admission_no_uq ON students (tenant_id, admission_no);
-- 3) contract (later release, after code writes it always)
ALTER TABLE students ALTER COLUMN admission_no SET NOT NULL;
```

## Anti-patterns

- `find()` / `SELECT *` without projection in API code.
- Queries inside loops; `populate` of large arrays.
- Indexes on every field "just in case"; no index on FK/tenant filter columns.
- Unbounded arrays embedded in Mongo documents.
- `synchronize: true` or `prisma db push` in production.
- Dropping/renaming columns in the same deploy as code changes.
- String-built SQL or `find(req.query)`.
- Creating DB clients per request; ignoring pool limits in serverless.
- Floats for money; `timestamp` without time zone.

## Checklist

- [ ] Existing schema, migrations and indexes inspected.
- [ ] Projections/selects on all list/lookup queries; `.lean()` where appropriate.
- [ ] No N+1; joins/populates limited to displayed fields.
- [ ] Each new index justified by a query pattern; verified with EXPLAIN when important.
- [ ] Pagination with deterministic order; keyset for large/infinite lists.
- [ ] Constraints/validation enforce invariants; tenant scoping present.
- [ ] Multi-write invariants in transactions.
- [ ] Migrations are backward compatible (expand/contract); data backfill planned.
- [ ] Inputs parameterized/sanitized; no raw query objects from clients.
