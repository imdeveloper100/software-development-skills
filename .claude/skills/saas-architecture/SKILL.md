---
name: saas-architecture
description: SaaS product architecture — multi-tenancy model selection, tenant isolation and tenant-aware queries, organizations/users/memberships, roles and permissions per tenant, subscriptions and plans, entitlements and usage limits, feature flags, billing-provider boundaries, audit logs and tenant configuration. Use when building multi-tenant/organization features, plans/billing, feature gating, audit logging, or reviewing tenant data isolation.
---

# SaaS Architecture

## Purpose

Keep every tenant's data isolated and every plan rule enforced in one place — while implementing only the SaaS capabilities the product actually needs right now.

## Rules

### R1. Only build what the product requires
Before adding billing, feature flags, usage metering, or audit logs, confirm a requirement exists. A single-tenant internal tool doesn't need multi-tenancy. But **if** the product is or will soon be multi-tenant, add `tenantId` from day one — retrofitting it is expensive.

### R2. Choose a tenancy model explicitly
| Model | When | Cost |
|---|---|---|
| **Shared DB, shared schema, `tenant_id` column** (default) | Most B2B SaaS, many small/medium tenants | Isolation is enforced by code (and RLS) — must be rigorous |
| Shared DB, schema-per-tenant (Postgres) | Moderate number of tenants needing stronger separation/customization | Migrations × tenants; connection/search_path handling |
| Database-per-tenant | Few large/enterprise tenants, compliance/data-residency demands | Operational overhead; fleet migrations |
Default to shared schema + `tenant_id`. Hybrid (dedicated DB for enterprise) can be added later if the data-access layer resolves the connection per tenant.

### R3. Tenant isolation rules (shared schema)
- Every tenant-owned table/collection has a non-null, indexed `tenantId`, leading composite indexes.
- `tenantId` comes from the **authenticated context** (token/session membership) — never from body/query. For users in multiple orgs, the active org is selected (header/route `/{orgSlug}`) and **verified against memberships** server-side.
- Every query on tenant data filters by `tenantId` — enforce centrally where possible:
  - Prisma: client extension adding `tenantId` to where/data for tenant models.
  - Mongoose: plugin/middleware adding `tenantId` to queries.
  - Postgres: **Row-Level Security** with `SET app.tenant_id` per transaction as defense-in-depth for higher-risk products.
  - NestJS: tenant context via `nestjs-cls` (AsyncLocalStorage), not request-scoped providers everywhere.
- Unique constraints are per tenant: `UNIQUE (tenant_id, email)`.
- Cross-tenant access returns 404.
- Background jobs, caches, file storage keys, search indexes and logs all carry `tenantId` (`s3://bucket/tenants/{tenantId}/...`, cache key prefix `t:{tenantId}:`).
- Platform super-admin access is explicit, audited, and goes through separate routes/guards.

### R4. Identity model
```
User (global identity: email, password/OAuth)  ──<  Membership (userId, tenantId, roleId, status)  >──  Tenant/Organization
                                                                                                        │
                                                                                         Subscription, Settings, Invitations
```
- A user can belong to multiple tenants unless the product says otherwise.
- Invitations: tokenized, expiring, single-use; accepting creates a membership.
- Roles are per membership. Predefined roles first; tenant-customizable roles only when required (then roles + permissions in DB).

### R5. Plans, entitlements, limits
- Separate **billing** (provider: Stripe, Paddle, LemonSqueezy) from **entitlements** (what the tenant may use).
- Store on the tenant: `planId`, `subscriptionStatus`, `currentPeriodEnd`, and derive entitlements from a plan config:
```ts
// config/plans.ts
export const PLANS = {
  free:     { maxStudents: 50,   features: { reports: false, apiAccess: false } },
  pro:      { maxStudents: 1000, features: { reports: true,  apiAccess: false } },
  business: { maxStudents: null, features: { reports: true,  apiAccess: true  } },
} as const;
```
- One `EntitlementsService` answers `can(tenant, 'reports')` and `checkLimit(tenant, 'maxStudents', currentCount)`. No `if (tenant.plan === 'pro')` scattered in code.
- Enforce limits server-side at the write path (in the same transaction or with a safe count); show remaining usage in UI.
- Usage metering only if billing is usage-based: record events (`usage_events`) and aggregate; don't count on the fly from big tables for every request.

### R6. Billing boundary
- Provider is the source of truth for payments; sync via **verified, idempotent webhooks** (signature check, dedupe by event ID, handle out-of-order events by checking timestamps/status).
- Never trust client-side "payment succeeded" redirects for provisioning.
- Keep provider-specific code in one `billing` module behind your own types (`SubscriptionStatus`), so switching providers or adding manual invoicing doesn't touch features.
- Define behavior for `past_due`, `canceled`, trial end: grace period, read-only mode, or lockout — explicitly.

### R7. Feature flags
- Distinguish: **entitlements** (plan-based, permanent) vs **release flags** (temporary rollout) vs **ops kill-switches**.
- Start simple: config/env or a `feature_flags` table with per-tenant overrides. Adopt a flag service (Unleash, GrowthBook, LaunchDarkly) only when you need targeting/experiments at scale.
- Evaluate on the server; send resolved flags to the client for UX.
- Remove release flags after rollout (track them).

### R8. Audit logs (when required: B2B, compliance, admin actions, money)
- Append-only `audit_logs`: `tenantId`, `actorUserId`, `action` (`student.deleted`), `entityType`, `entityId`, `changes` (diff, redacted), `ip`, `userAgent`, `createdAt`.
- Write from services (explicit calls or domain events), not DB triggers that lack actor context.
- Never store secrets/passwords in diffs. Index `(tenantId, createdAt)` and `(tenantId, entityType, entityId)`. Define retention.
- Audit logs are distinct from application logs.

### R9. Tenant configuration
Settings in a typed `tenant_settings` (JSONB/embedded doc validated by a schema with defaults) — not dozens of nullable columns, not untyped blobs. Cache per tenant with invalidation on update.

### R10. Noisy neighbors & scale path
Per-tenant rate limits on expensive endpoints; per-tenant job concurrency in queues; monitor top tenants by usage. Path to scale: index by tenant → read replicas → move large tenants to dedicated DB → shard by tenant. Don't build these until metrics demand it (see `system-design`).

## Implementation Guidelines

1. Confirm tenancy model and how the active tenant is resolved.
2. New tenant-owned entity: `tenantId` column + index + central scoping + tests for cross-tenant access.
3. Plan-gated feature: add to plan config → check via `EntitlementsService` server-side → expose in `/me` or `/tenant/entitlements` for UI.
4. Billing changes: only via webhook handlers in the billing module.
5. Sensitive admin actions: add audit log entries.

## Examples

**Prisma tenant scoping extension (sketch)**
```ts
export function tenantClient(prisma: PrismaClient, tenantId: string) {
  return prisma.$extends({
    query: {
      $allModels: {
        async $allOperations({ model, operation, args, query }) {
          if (!TENANT_MODELS.has(model)) return query(args);
          if (['findMany','findFirst','count','updateMany','deleteMany','aggregate'].includes(operation))
            args.where = { ...args.where, tenantId };
          if (operation === 'create') args.data = { ...args.data, tenantId };
          return query(args);
        },
      },
    },
  });
}
// Note: findUnique/update/delete by id must use compound unique (id+tenantId) or be converted to *Many/First variants.
```

**Limit check at write**
```ts
async createStudent(ctx: TenantCtx, dto: CreateStudentDto) {
  const count = await this.db.student.count({ where: { tenantId: ctx.tenantId, deletedAt: null } });
  this.entitlements.assertWithinLimit(ctx.tenant, 'maxStudents', count + 1); // throws 403 PLAN_LIMIT_REACHED
  ...
}
```

## Anti-patterns

- `tenantId` accepted from request body or query.
- Queries on tenant tables without tenant filter (relying on "the UI only shows their IDs").
- `if (plan === 'pro')` across controllers and React components.
- Provisioning access on client redirect after checkout.
- Global unique constraints on fields that should be unique per tenant.
- Cache keys/S3 paths without tenant prefix.
- Building metering, flags, audit, custom roles "for later" with no requirement.

## Checklist

- [ ] Tenancy model explicit; active tenant resolved from auth + membership.
- [ ] Every tenant-owned entity has indexed `tenantId`; scoping enforced centrally.
- [ ] Cross-tenant access tests (expect 404).
- [ ] Jobs, cache keys, files, logs include tenant.
- [ ] Plan rules via one entitlements service; limits enforced server-side.
- [ ] Billing state only from verified, idempotent webhooks.
- [ ] Flags categorized; release flags tracked for removal.
- [ ] Audit logs for sensitive actions (when required), redacted.
- [ ] Only SaaS features with a real requirement were added.
