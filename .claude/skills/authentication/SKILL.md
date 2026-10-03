---
name: authentication
description: Authentication and authorization rules — choosing sessions vs JWT, access/refresh token lifecycle and rotation, token storage (httpOnly cookies), password hashing, OAuth/OIDC, logout and revocation, RBAC and permission design, server-side guards, frontend route protection and auth state, CSRF and security headers. Use when implementing or changing login, signup, tokens, sessions, roles, permissions, protected routes, or anything that decides who can access what.
---

# Authentication & Authorization

## Purpose

Prove who the user is (authentication) and enforce what they may do (authorization) **on the server**, with secure defaults and no home-grown crypto.

## Rules

### R1. Prefer proven solutions
Use the project's existing auth. For new projects prefer an established library/provider (Auth.js/NextAuth, Better Auth, Passport + `@nestjs/passport`, Clerk/Auth0/Cognito) over a custom implementation, unless requirements demand custom. Never invent token formats or crypto.

### R2. Sessions vs JWT — decide explicitly
| Situation | Choose |
|---|---|
| Web app + backend on same site/domain | **Server sessions** (httpOnly cookie + store) or short-lived JWT in httpOnly cookie |
| Separate SPA + API on different subdomains | httpOnly cookie (with `Domain`, `SameSite`, CORS credentials) or access token in memory + refresh token in httpOnly cookie |
| Mobile apps / third-party API clients | Bearer access token + refresh token (secure storage on device) |
| Need instant revocation (ban user, change role) | Sessions, or short access TTL (5–15 min) + server-side refresh token store |

### R3. Token lifecycle (when using JWT)
- Access token: short TTL (5–15 min), minimal claims (`sub`, `tenantId`, `roles`/`permissionsVersion`, `jti`) — no PII, no secrets.
- Refresh token: longer TTL (7–30 days), **stored hashed in DB**, rotated on every use; reuse of an old refresh token → revoke the whole token family (theft detection).
- Sign with a strong secret (≥ 32 random bytes) or asymmetric keys (RS256/EdDSA) when other services verify tokens. Verify `alg`, `exp`, `iss`, `aud`.
- Separate secrets for access and refresh tokens.

### R4. Token storage (browser)
- **Do**: httpOnly + Secure + `SameSite=Lax` (or `Strict`) cookies for refresh tokens / session IDs; access token in memory (or httpOnly cookie).
- **Don't**: store refresh tokens or long-lived tokens in `localStorage`/`sessionStorage` (any XSS steals them).
- If the project currently uses localStorage, flag the risk; don't silently rewrite auth in an unrelated task.

### R5. Passwords
- Hash with **argon2id** (preferred) or **bcrypt** (cost ≥ 12). Never MD5/SHA-x, never reversible encryption.
- Min length 8 (prefer 12+), max ~128; allow paste and password managers; optionally check against breached lists.
- Generic login error: "Invalid email or password". Same response timing for unknown users where feasible.
- Rate-limit + lockout/backoff on login, reset, OTP endpoints.
- Password reset: single-use, random token (hashed in DB), 15–60 min expiry, invalidate sessions after reset.

### R6. OAuth / OIDC
Use Authorization Code + PKCE; validate `state` (and `nonce` for OIDC); exact redirect URI whitelist; link accounts by verified email only; store provider tokens encrypted if you must store them.

### R7. Logout & revocation
Logout clears cookies **and** revokes the server-side session/refresh token. Provide "log out all devices" when sessions matter (delete all refresh tokens for user). On password change/role change, invalidate existing sessions or bump a `tokenVersion`.

### R8. Authorization is server-side, always
- Every protected endpoint/action/handler checks: (1) authenticated, (2) has permission, (3) owns/can access **this** resource (tenant, ownership, assignment).
- Frontend checks (hide buttons, route guards) are UX only.
- Missing permission → 403. Resource not in user's tenant/scope → 404.
- Check ownership in the **query** (`where: { id, tenantId }`) rather than fetching then comparing — prevents IDOR and races.

### R9. RBAC design
- Roles map to **permissions**; code checks permissions (`students.update`), not role names (`if role === 'admin'`), except for a super-admin bypass handled centrally.
- Permission naming: `resource.action` (`fees.read`, `fees.refund`).
- Keep the role→permission map in one place (DB table if roles are tenant-configurable; constant if fixed). The frontend gets the user's effective permissions from `/me` for UX.
- Add ABAC conditions (own records only, same department) in services where needed; don't create a role explosion (`teacher_own_class_editor`).

### R10. Frontend auth state
- Single source: `/me` (or session) query cached in the server-state library, or the framework's session hook. Don't duplicate into multiple contexts/stores.
- Axios/fetch interceptor handles 401 → one refresh attempt (single-flight; queue concurrent requests) → retry → on failure, clear cache and redirect to login with `returnTo`.
- On logout: clear query cache (`queryClient.clear()`), in-memory tokens, and any user-scoped storage.
- Route protection: protected layout/route wrapper (SPA) or middleware + server checks (Next.js). Avoid flashing protected content before auth resolves.

### R11. CSRF
Needed when auth relies on cookies sent automatically. Mitigate with `SameSite=Lax/Strict` cookies **plus** CSRF tokens (double-submit or synchronizer) for state-changing requests if cross-site forms are possible or `SameSite=None` is used; verify `Origin` header on mutations. Bearer-token-in-header APIs aren't CSRF-prone (but are XSS-sensitive). Next.js Server Actions check Origin by default — keep `allowedOrigins` narrow.

### R12. Security headers
`helmet()` (or equivalent): HSTS, `X-Content-Type-Options`, frame-ancestors/`X-Frame-Options`, Referrer-Policy, and a CSP where feasible (start report-only). CORS: explicit origin allowlist; `credentials: true` only with explicit origins (never `*`).

### R13. Secrets
Auth secrets/keys only in env/secret manager; never in code, images, logs or client bundles. Rotate when exposed.

## Implementation Guidelines

1. Identify current auth mechanism, token storage, guards, and how `/me`/session is consumed.
2. For new protected endpoints: add guard + permission + ownership/tenant-scoped query.
3. For new roles/permissions: update the single mapping, the seed/migration, and the frontend permission constants (UX).
4. Test: unauthenticated 401, wrong permission 403, other tenant's resource 404, happy path.

## Examples

**Ownership in the query (prevents IDOR)**
```ts
// ❌ fetch by id, forgot tenant check
const invoice = await prisma.invoice.findUnique({ where: { id } });

// ✅
const invoice = await prisma.invoice.findFirst({ where: { id, tenantId: user.tenantId } });
if (!invoice) throw new NotFoundException();
```

**Refresh rotation sketch**
```ts
async refresh(rawToken: string) {
  const record = await this.tokens.findByHash(sha256(rawToken));
  if (!record || record.expiresAt < new Date()) throw new UnauthorizedException();
  if (record.revokedAt) { await this.tokens.revokeFamily(record.familyId); throw new UnauthorizedException(); } // reuse detected
  await this.tokens.revoke(record.id);
  const next = await this.tokens.issue(record.userId, record.familyId);
  return { accessToken: this.signAccess(record.userId), refreshToken: next.raw }; // refresh set as httpOnly cookie by controller
}
```

**Cookie settings**
```ts
res.cookie('refresh_token', token, {
  httpOnly: true, secure: true, sameSite: 'lax', path: '/api/auth/refresh', maxAge: 30 * 24 * 3600 * 1000,
});
```

## Anti-patterns

- Authorization only in React (hidden buttons) with open API endpoints.
- `if (user.role === 'admin')` scattered across the codebase.
- JWTs with 30-day lifetime and no revocation; refresh tokens in localStorage.
- `findById(req.params.id)` without tenant/ownership scope.
- Detailed login errors ("email not found").
- `cors({ origin: '*', credentials: true })`.
- Secrets in `NEXT_PUBLIC_`/`VITE_` vars or committed `.env`.
- Logging tokens or Authorization headers.

## Checklist

- [ ] Auth mechanism chosen deliberately and consistent with the project.
- [ ] Passwords hashed with argon2id/bcrypt; auth endpoints rate-limited.
- [ ] Tokens: short access TTL, rotated hashed refresh tokens, httpOnly cookies.
- [ ] Logout revokes server-side; role/password change invalidates sessions.
- [ ] Every protected route: authn + permission + tenant/ownership-scoped query.
- [ ] Permissions centralized; frontend checks are UX only.
- [ ] CSRF handled for cookie auth; CORS allowlist; security headers set.
- [ ] Tests for 401/403/404/happy path.
