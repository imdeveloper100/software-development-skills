---
name: api-design
description: REST API design rules — resource naming, HTTP methods and status codes, request/response DTOs, slim purpose-specific responses, lookup endpoints for dropdowns/autocomplete, pagination (offset vs cursor), filtering/sorting/searching, bulk operations, standard error envelope, versioning, rate limiting and idempotency. Use when designing, adding, changing or reviewing any API endpoint or when a frontend consumes an endpoint that returns more data than it needs.
---

# API Design

## Purpose

APIs that are predictable, secure and **slim**: each endpoint returns what its consumers need — no more — in a consistent shape.

## Rules

### R1. Resource naming
- Plural nouns, kebab-case: `/students`, `/fee-invoices`, `/students/{id}`.
- Nesting only for true ownership, max one level: `/classes/{classId}/students`. Otherwise filter: `/students?classId=…`.
- Actions that aren't CRUD: sub-resource or verb sub-path on the resource: `POST /invoices/{id}/send`, `POST /subscriptions/{id}/cancel`. Avoid `/getStudents`, `/createStudent`.
- JSON fields: `camelCase` (or follow existing API). Dates as ISO-8601 UTC strings. Money as integer minor units (`amountCents`) or decimal strings — never floats. IDs as strings.

### R2. Methods & status codes
| Operation | Method | Success |
|---|---|---|
| List / read | GET | 200 |
| Create | POST | 201 (+ created resource or `{ id }`; `Location` header optional) |
| Full replace | PUT | 200 |
| Partial update | PATCH | 200 (updated resource) |
| Delete | DELETE | 204 |
| Async accepted (job) | POST | 202 + job id/status URL |
| Action without body result | POST | 200 / 204 |

Errors: 400 malformed/validation (or 422 if the project uses it — be consistent), 401 not authenticated, 403 authenticated but not allowed, 404 not found **or not visible to this tenant/user**, 409 conflict (duplicate, invalid state transition, version mismatch), 413 payload too large, 429 rate limited, 500 unexpected. Never 200 with `{ success: false }`.

### R3. Validate every input
Body, query, params validated and bounded (see `backend-architecture` R4). Unknown fields rejected/stripped. Server-owned fields (`id`, `tenantId`, `role`, `createdAt`, `ownerId`) never accepted from clients.

### R4. Slim, purpose-specific responses
- **List** endpoints return list-item DTOs (fields the table/card shows), not full entities with nested relations.
- **Detail** endpoints return the detail DTO; heavy relations (history, attachments, large child lists) get their own paginated endpoints.
- No nested arrays that can grow unbounded (`students: [...]` inside a class).
- Never return: password hashes, tokens, internal flags, other tenants' data, soft-deleted records (unless requested by authorized admin).
- Use DB projections/selects so over-fetching doesn't even reach the app layer (see `database`).
- If multiple consumers need different shapes, prefer dedicated endpoints (`/lookup`, `/summary`) over one endpoint with every field. A `fields=` sparse-fieldset param is acceptable for public/general APIs, but must be whitelisted.

### R5. Lookup endpoints (dropdowns, autocomplete, selectors, suggestions)
- Path: `GET /{resource}/lookup`.
- Response: array of `{ id, name }` (or `{ value, label }` if the frontend standard uses that) + only extra fields the selector **displays** (e.g., `code`, `avatarUrl`).
- Params: `search` (min 1–2 chars for large sets, prefix/contains match), `limit` (default 20, max 50), optional scoping filters (`classId`, `status=active`), optional `ids=` to resolve already-selected values for edit forms.
- Only active/selectable records by default.
- Indexed search field (see `database`); sorted by name or relevance.
- Small static sets (< ~200 items, rarely change: departments, roles) may return all items without `search`, and should be cached on the client with long `staleTime`.
- **Do not reuse** the full `/users` list endpoint for a dropdown because it exists.

```
GET /users/lookup?search=awa&limit=20
200 [ { "id": "123", "name": "Awais Rehman" } ]
```

### R6. Pagination
Any collection that can grow without bound is paginated. Tiny bounded sets (enums, < ~100 config rows) are not.
- **Offset** (`page`, `limit`) — admin tables needing page numbers/jump-to-page, moderate data size.
  ```json
  { "items": [...], "meta": { "page": 2, "limit": 20, "total": 534, "totalPages": 27, "hasNext": true, "hasPrevious": true } }
  ```
- **Cursor** (`cursor`, `limit`) — feeds, infinite scroll, large/fast-changing tables, exports, sync. Cursor = opaque base64 of the sort key + id (tie-breaker).
  ```json
  { "items": [...], "meta": { "limit": 20, "nextCursor": "eyJjcmVhdGVkQXQiOi4uLn0", "hasNext": true } }
  ```
- Defaults: `limit=20`, max 100 (enforced server-side). Always a deterministic sort (add `id` as tie-breaker).
- `total` requires a `COUNT` — skip or approximate it for very large tables if it's expensive and the UI doesn't need it.

### R7. Filtering, sorting, searching
- Filters as explicit query params: `?status=active&classId=…&createdFrom=2026-01-01`.
- Sorting: `?sort=createdAt` / `?sort=-createdAt` (or `sortBy`/`sortOrder` if established). **Whitelist** sortable fields (and only indexed ones for large tables).
- Search: `?search=` on defined fields; escape regex input in Mongo; use full-text/trigram indexes for large sets.
- Never pass raw query objects from the client into the DB (`find(req.query)` = NoSQL injection).

### R8. Bulk operations
`POST /students/bulk` (create), `PATCH /students/bulk` (update), `POST /students/bulk-delete` with `{ ids: [] }`. Cap batch size (e.g., 100–500). Decide and document atomic vs partial success; for partial, return per-item results:
```json
{ "succeeded": ["id1","id2"], "failed": [{ "id": "id3", "code": "NOT_FOUND" }] }
```
Large imports → async job (202) per `backend-architecture` R11.

### R9. Error envelope (one format everywhere)
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [ { "field": "email", "message": "must be a valid email" } ],
    "requestId": "req_8f2a…"
  }
}
```
- `code`: stable machine-readable string the frontend can switch on (`CLASS_FULL`, `EMAIL_TAKEN`).
- `message`: safe for display; no stack traces, SQL, internal IDs of other tenants.
- Follow the existing envelope if the project already has one (e.g., RFC 7807 `application/problem+json`).

### R10. Versioning & compatibility
- Version prefix `/api/v1` if the API has external/mobile consumers or is public. Internal APIs consumed only by your own web app may stay unversioned but must still evolve compatibly.
- Non-breaking: add optional fields/params, new endpoints. Breaking: remove/rename fields, change types/semantics, make optional required → needs a new version or coordinated deploy of all consumers.
- Search the frontend for consumers before changing a response.

### R11. Auth on every endpoint
Default-deny: every route requires auth unless explicitly public. Authorization checks role/permission **and** resource ownership/tenant (see `authentication`, `saas-architecture`). Return 404 instead of 403 for resources in other tenants to avoid enumeration.

### R12. Rate limiting (where appropriate)
Mandatory for: login, signup, password reset, OTP, public search, expensive endpoints (exports, reports), webhooks receivers. Per IP and/or per user/tenant. Return 429 with `Retry-After`. In-memory limiter for single instance; Redis-backed when running multiple instances.

### R13. Idempotency (where appropriate)
For POSTs where duplicates are harmful (payments, orders, invoice creation, external webhooks): accept `Idempotency-Key` header; store key + request hash + response for 24h; replay stored response on retry. For webhooks, dedupe by provider event ID. PUT/DELETE must be idempotent by design.

## Implementation Guidelines

1. Identify the **consumer** (which screen/component/service) and list the fields it actually uses.
2. Choose: existing endpoint fits? → use it. Over-returns? → add `/lookup`, `/summary`, or a slimmer list DTO.
3. Define request DTO (validated, bounded) and response DTO.
4. Decide pagination type and limits; whitelist filters/sorts.
5. Map errors to the standard envelope with codes.
6. Check consumers before changing existing contracts; document in OpenAPI if used.

## Examples

**Slim list vs over-fetch**
```ts
// ❌ GET /classes returns full documents with students[], teachers[], subjects[]
return ClassModel.find({ tenantId }).populate('students teachers subjects');

// ✅ GET /classes (table)
return prisma.class.findMany({
  where: { tenantId }, skip, take: limit, orderBy: [{ name: 'asc' }, { id: 'asc' }],
  select: { id: true, name: true, section: true, _count: { select: { students: true } } },
});

// ✅ GET /classes/lookup (dropdown)
return prisma.class.findMany({
  where: { tenantId, isActive: true, ...(search && { name: { contains: search, mode: 'insensitive' } }) },
  select: { id: true, name: true }, orderBy: { name: 'asc' }, take: Math.min(limit, 50),
});
```

## Anti-patterns

- `GET /getAllUsers` returning thousands of full user objects.
- 200 OK with `{ success: false, error: '...' }`.
- Different error shapes per controller.
- Unbounded `limit` (`?limit=100000`).
- Sorting by arbitrary client-provided field names.
- Returning `passwordHash`, `refreshToken`, or `__v`/internal fields.
- Accepting `tenantId`/`role` in request bodies.
- Breaking a response shape without checking consumers.
- Reusing heavy endpoints for dropdowns.

## Checklist

- [ ] Resource naming and methods follow conventions; correct status codes.
- [ ] All inputs validated; server-owned fields ignored/rejected.
- [ ] Response contains only fields consumers use; no sensitive fields; projections used.
- [ ] Selector UIs served by `/lookup` (id + label, search, limit).
- [ ] Unbounded collections paginated with enforced max limit and deterministic sort.
- [ ] Filters/sorts whitelisted; search input escaped/indexed.
- [ ] Standard error envelope with stable codes.
- [ ] Auth + authz (incl. tenant/ownership) on the endpoint.
- [ ] Rate limiting / idempotency considered for sensitive or money-related endpoints.
- [ ] Contract changes are backward compatible or versioned; consumers checked.
