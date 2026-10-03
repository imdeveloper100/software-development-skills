---
name: docker
description: Docker and Docker Compose rules for Node, NestJS, React (Vite) and Next.js apps — multi-stage production Dockerfiles, .dockerignore, layer caching, small non-root images, env vars and secrets at runtime (not baked), build-time public vars, health checks, dev vs prod compose, networking, volumes and service dependencies. Use when creating or modifying Dockerfiles, compose files, container configs, or debugging container builds.
---

# Docker

## Purpose

Reproducible, small, secure images that build fast from cache, and a compose setup that gives a one-command local environment — without dev tooling or secrets in production images.

## Rules

### R1. Multi-stage builds for production
Stages: `deps` (install with lockfile) → `build` (compile) → `runner` (only runtime artifacts + production deps). The final image contains no source TS, no devDependencies, no build tools, no `.git`.

### R2. Base image
- `node:<LTS>-alpine` or `node:<LTS>-bookworm-slim` (use slim if native modules like `bcrypt`/`sharp`/Prisma engines give trouble on Alpine). Pin the major/minor version (`node:22-alpine`, not `node:latest`); pin by digest for strict reproducibility if the team wants it.
- Match the Node version in `package.json` `engines`/`.nvmrc`.

### R3. Layer caching
Copy manifests first, install, then copy source:
```dockerfile
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
```
Use `npm ci` / `pnpm install --frozen-lockfile` / `yarn install --immutable`. Use BuildKit cache mounts (`--mount=type=cache,target=/root/.local/share/pnpm/store`) when available.

### R4. `.dockerignore` is mandatory
```
node_modules
dist
.next
coverage
.git
.env*
!.env.example
Dockerfile*
docker-compose*
*.log
.vscode
.idea
```

### R5. Secrets & configuration
- No secrets in `Dockerfile`, `ARG`, `ENV`, or copied `.env` files — they persist in layers/history.
- Runtime config via environment variables injected by compose/orchestrator/secret manager.
- Build-time secrets (private registry token) via BuildKit `--secret`, never `ARG`.
- Public build-time vars (`NEXT_PUBLIC_*`, `VITE_*`) are baked into the bundle at build: document that the image is environment-specific, or use runtime config injection (e.g., serve a `/config.js`/`env.js` generated at container start) if one image must be promoted across environments.

### R6. Runtime hardening
- Run as non-root (`USER node` in official images).
- `NODE_ENV=production`.
- Run `node dist/main.js` directly (not `npm start`) so signals reach Node; handle `SIGTERM` for graceful shutdown (Nest: `enableShutdownHooks()`; Express: close server + DB pool). Use `--init`/`tini` if you spawn child processes.
- Expose only the app port. No SSH, no package managers needed at runtime.

### R7. Health checks
- App exposes `GET /health` (liveness: process up) and optionally `/health/ready` (readiness: DB reachable). Keep them cheap and unauthenticated; don't leak details.
- `HEALTHCHECK` in Dockerfile or `healthcheck:` in compose (prefer compose/orchestrator-level in AWS ECS/K8s since they define their own).
- Alpine images lack `curl` by default — use `wget -qO-` or a tiny Node script.

### R8. Framework specifics
- **NestJS/Express (TS)**: build with `tsc`/`nest build` in build stage; `pnpm prune --prod` (or reinstall `--prod`) for runner; copy `dist` + prod `node_modules` + `package.json`. Prisma: run `prisma generate` in build, copy generated client and `prisma/` (migrations) if migrations run from the image; ensure the correct `binaryTargets`.
- **Next.js**: set `output: 'standalone'` in `next.config`; runner copies `.next/standalone`, `.next/static` → `.next/static`, and `public`. Run `node server.js`. Set `HOSTNAME=0.0.0.0`.
- **React SPA (Vite)**: build stage then serve `dist` with `nginx:alpine` (with SPA fallback `try_files $uri /index.html;`, long cache for hashed assets) — or upload to S3+CloudFront and skip containers entirely.

### R9. Migrations
Don't run migrations automatically on every container start when multiple replicas start concurrently — run them as a separate one-off step/job in the deploy pipeline (`prisma migrate deploy`), or ensure the tool takes a lock.

### R10. Compose for local development
- `docker-compose.yml` (or `compose.yaml`) for infra the app needs: DB, Redis (if used), Mailpit, MinIO (S3-compatible) — only what the app actually uses.
- Dev app container (optional): bind-mount source, anonymous volume for `node_modules`, run dev server; or run the app on the host and only infra in compose (often simpler/faster).
- Named volumes for DB data; healthchecks on DBs and `depends_on: { db: { condition: service_healthy } }`.
- Services talk via service name on the compose network (`postgres://app:app@db:5432/app`), not `localhost`.
- Publish only needed ports; bind DB ports to `127.0.0.1` locally.
- Separate prod-like overrides in `compose.prod.yml` if used; never dev bind mounts in prod.

### R11. Image size & scanning
Check `docker images` size after changes; typical Node API runtime should be ~150–300 MB or less on Alpine/slim. Scan images (`docker scout cves`, Trivy) in CI for high/critical CVEs.

## Implementation Guidelines

1. Inspect existing Dockerfiles/compose and package manager; keep the same base and conventions.
2. Write/adjust multi-stage Dockerfile + `.dockerignore`.
3. Verify: `docker build -t app .`, `docker run --env-file .env.local -p 3000:3000 app`, hit `/health`.
4. `docker compose config` to validate compose; `docker compose up` for local flow.
5. Report image size and any env vars that must be provided at runtime.

## Examples

**NestJS / Node API**
```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS base
WORKDIR /app
RUN corepack enable

FROM base AS deps
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store pnpm install --frozen-lockfile

FROM deps AS build
COPY . .
RUN pnpm prisma generate && pnpm build && pnpm prune --prod

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build --chown=node:node /app/package.json ./
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --from=build --chown=node:node /app/prisma ./prisma
USER node
EXPOSE 4000
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s CMD wget -qO- http://127.0.0.1:4000/health || exit 1
CMD ["node", "dist/main.js"]
```

**Next.js standalone**
```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_API_URL            # public, baked at build time
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production HOSTNAME=0.0.0.0 PORT=3000
COPY --from=build --chown=node:node /app/public ./public
COPY --from=build --chown=node:node /app/.next/standalone ./
COPY --from=build --chown=node:node /app/.next/static ./.next/static
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

**Local infra compose**
```yaml
services:
  db:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: app }
    ports: ["127.0.0.1:5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U app"], interval: 5s, retries: 10 }
  api:
    build: { context: ., target: deps }
    command: pnpm start:dev
    env_file: .env
    environment: { DATABASE_URL: postgres://app:app@db:5432/app }
    volumes: [".:/app", "/app/node_modules"]
    ports: ["4000:4000"]
    depends_on: { db: { condition: service_healthy } }
volumes: { pgdata: {} }
```
(Local-only credentials are fine in dev compose; production values come from the secret manager.)

## Anti-patterns

- Single-stage image with devDependencies, source and build tools.
- `COPY . .` before `npm install` (busts cache every change).
- Missing `.dockerignore` (copies `node_modules`, `.env`, `.git`).
- `ENV JWT_SECRET=...` or `COPY .env .` in Dockerfile.
- Running as root; `CMD npm start`.
- `node:latest`.
- Migrations run concurrently by every replica at boot.
- `localhost` used between containers.
- Adding Redis/Kafka containers the app doesn't use.

## Checklist

- [ ] Multi-stage; runner has only runtime artifacts + prod deps.
- [ ] Pinned base image matching project Node version.
- [ ] Manifests copied before source; frozen-lockfile install.
- [ ] `.dockerignore` excludes node_modules, .env, .git, build outputs.
- [ ] No secrets in image; runtime env documented; public build vars understood.
- [ ] Non-root, `NODE_ENV=production`, direct `node` CMD, graceful shutdown.
- [ ] Health endpoint + healthcheck.
- [ ] Compose: service-name networking, named volumes, healthy depends_on.
- [ ] `docker build` and run verified; image size reported.
