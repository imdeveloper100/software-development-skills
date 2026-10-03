---
name: security
description: Practical application-security rules (OWASP-aligned) for Node/React/Next apps — input validation, SQL/NoSQL injection, XSS, CSRF, CORS, IDOR/broken access control, rate limiting, file uploads, SSRF, secrets handling, sensitive logging, error leakage and dependency vulnerabilities. Use when handling user input, files, URLs, HTML rendering, external requests, secrets, or when reviewing code for security issues.
---

# Security

## Purpose

Prevent the vulnerabilities that actually hit web apps, with controls proportionate to the application. Authentication/authorization details live in `authentication`; this skill covers everything around them.

## Rules

### R1. Validate all untrusted input at the boundary
Untrusted = request body/query/params/headers/cookies, uploaded files, webhook payloads, third-party API responses, URL params in the frontend. Validate type, length, format, range, and allowed values (allowlist). Reject unknown fields.

### R2. Injection
- **SQL**: parameterized queries/ORM only. No string concatenation into SQL; no `$queryRawUnsafe`/`sequelize.query` with interpolated user data. Whitelist identifiers (sort columns) — they can't be parameterized.
- **NoSQL**: never pass `req.body`/`req.query` objects directly into filters; cast to primitives (`String(email)`), enable Mongoose `sanitizeFilter` or `express-mongo-sanitize`; escape regex input.
- **Command**: avoid `exec` with user input; use `execFile`/`spawn` with argument arrays and allowlists.
- **Template/path**: never build file paths from user input without `path.resolve` + prefix check against an allowed directory.

### R3. XSS
- React escapes by default. `dangerouslySetInnerHTML` only with sanitized HTML (`DOMPurify`/`isomorphic-dompurify`) — and only when rich content is a requirement.
- Never put user input into `href`/`src` without validating protocol (`http:`/`https:`/`mailto:` only — block `javascript:`).
- JSON-LD/inline scripts: escape `<` when serializing.
- Add CSP when feasible (start `Content-Security-Policy-Report-Only`).

### R4. Broken access control / IDOR
Every query for a resource by ID is scoped to tenant/owner (`where: { id, tenantId }`). Never trust IDs in the body for ownership (`userId`, `tenantId`). Test with a second user/tenant. (Details: `authentication` R8.)

### R5. CSRF & CORS
- CSRF: see `authentication` R11.
- CORS: allowlist exact origins from config per environment. `credentials: true` never with `*`. Don't reflect arbitrary `Origin`. CORS is not an auth mechanism — server-to-server calls ignore it.

### R6. Rate limiting & abuse
Rate-limit auth, OTP, password reset, signup, contact forms, public search, export/report endpoints, and anything that sends email/SMS or costs money. Limit body size (`express.json({ limit: '1mb' })`), pagination `limit`, bulk sizes, and query complexity.

### R7. File uploads
- Allowlist extensions **and** verify MIME by magic bytes (`file-type`), not just `Content-Type`.
- Enforce size limits at the parser (multer `limits.fileSize`) and at the proxy.
- Store outside the web root, ideally object storage (S3) with **private** ACL; serve via short-lived signed URLs.
- Generate server-side file names (UUID); never use the client filename as a path.
- For direct browser→S3 uploads use presigned POST with content-type and size conditions.
- Images: re-encode/strip metadata if publicly displayed; SVG uploads treated as HTML (XSS) — disallow or sanitize.
- Malware scanning when files are shared with other users (e.g., ClamAV, S3 malware scanning) — conditional on risk.

### R8. SSRF
When the server fetches a user-supplied URL (webhooks config, link previews, image import, PDF generation from URL):
- Allowlist schemes (`https:`) and, where possible, domains.
- Resolve DNS and block private/loopback/link-local ranges (`127.0.0.0/8`, `10/8`, `172.16/12`, `192.168/16`, `169.254.169.254` metadata, `::1`, `fc00::/7`), and re-check after redirects (or disable redirects).
- Timeouts and response size limits.
- On AWS, require IMDSv2.

### R9. Secrets
- Never in source, Git history, Docker images, client bundles, URLs, or logs.
- `.env*` in `.gitignore` (except `.env.example`). Use a secret manager (AWS Secrets Manager/SSM, GitHub Actions secrets, Doppler) in deployed environments.
- If a secret is committed: rotate it immediately (removing from history is not enough).
- Scan: enable GitHub secret scanning/push protection or `gitleaks` in CI.

### R10. Sensitive data & logging
- Don't log passwords, tokens, cookies, Authorization headers, OTPs, full card/bank numbers, national IDs, full request bodies on auth/payment routes. Configure logger redaction (`pino` `redact` paths).
- Return minimal PII in APIs; mask where possible (`****1234`).
- Encrypt sensitive fields at rest when required by the domain (e.g., provider tokens, national IDs).
- HTTPS everywhere; HSTS in production.

### R11. Error leakage
Production error responses contain code + safe message + requestId only. No stack traces, SQL, ORM messages, file paths, or framework versions (`app.disable('x-powered-by')`, helmet).

### R12. Dependencies
- Run `npm audit`/`pnpm audit` (or Dependabot/Renovate/Snyk) in CI; fix high/critical with available patches.
- Prefer maintained packages; avoid installing packages for trivial functions (supply-chain risk).
- Commit lockfiles; use `npm ci`/`pnpm install --frozen-lockfile` in CI and Docker.
- Review new dependencies' install scripts/popularity when adding.

### R13. Webhooks
Verify signatures (Stripe, GitHub, etc.) using the raw body and constant-time comparison; reject old timestamps; dedupe by event ID (idempotency).

### R14. Proportionality
Match controls to risk: an internal admin tool still needs authz, validation and secrets hygiene; malware scanning, field-level encryption and WAFs are for higher-risk data/products. State what you chose not to add and why.

## Implementation Guidelines

1. Identify trust boundaries touched by the change (input, files, outbound URLs, HTML rendering, secrets).
2. Apply the matching rules above.
3. Add tests for malicious input on critical paths (injection payloads, other-tenant IDs, oversized files).
4. Grep the diff for: `dangerouslySetInnerHTML`, `$queryRawUnsafe`, `exec(`, `eval(`, `new Function`, `find(req.`, `origin: '*'`, `console.log(req`, hardcoded keys.

## Examples

```ts
// NoSQL injection: { "email": { "$ne": null } } would match any user
// ❌
const user = await User.findOne({ email: req.body.email });
// ✅ (after DTO validation guarantees a string)
const user = await User.findOne({ email: String(dto.email).toLowerCase() });

// Upload limits & type check
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 5 * 1024 * 1024, files: 1 } });
const type = await fileTypeFromBuffer(file.buffer);
if (!type || !['image/png', 'image/jpeg', 'application/pdf'].includes(type.mime)) throw new BadRequestException('Unsupported file type');
await s3.putObject({ Bucket, Key: `tenants/${tenantId}/docs/${randomUUID()}.${type.ext}`, Body: file.buffer, ContentType: type.mime });

// Logger redaction
const logger = pino({ redact: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token'] });
```

## Anti-patterns

- Trusting client-side validation only.
- `find(req.query)`, string-built SQL, `eval` on input.
- Rendering CMS/user HTML without sanitizing.
- Public S3 buckets for user documents; using client file names as storage keys.
- Fetching arbitrary user URLs from the server without SSRF protection.
- `.env` committed; secrets in Dockerfiles or `NEXT_PUBLIC_`.
- Stack traces in production API responses.
- Ignoring `npm audit` criticals indefinitely.

## Checklist

- [ ] All untrusted inputs validated (type, length, allowlist) at the boundary.
- [ ] Queries parameterized; no raw objects from clients; regex escaped.
- [ ] No unsanitized HTML; URLs protocol-checked.
- [ ] Resource access scoped by tenant/owner.
- [ ] CORS allowlist; CSRF handled for cookie auth.
- [ ] Rate limits/body limits on sensitive or expensive endpoints.
- [ ] Uploads: size/type verified, private storage, generated names.
- [ ] Outbound user-supplied URLs SSRF-protected.
- [ ] No secrets in code/images/logs/bundles; logger redaction configured.
- [ ] Production errors don't leak internals; dependency audit clean of high/critical.
