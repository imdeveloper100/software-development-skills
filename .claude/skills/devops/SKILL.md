---
name: devops
description: CI/CD and operations rules — GitHub Actions pipelines (install, lint, typecheck, test, build, image), environment separation (dev/staging/prod), secrets management, deployment strategies, database migrations in deploys, rollback, health checks, structured logs, monitoring/alerting/error tracking, and pragmatic AWS deployment options (S3+CloudFront, App Runner/ECS Fargate, RDS, Secrets Manager). Use when creating or changing pipelines, deployment configs, environments, infrastructure choices or observability.
---

# DevOps

## Purpose

Every change goes through an automated, repeatable path to production that can be rolled back, observed and debugged — using the least infrastructure that does the job.

## Rules

### R1. Simplest viable infrastructure
Pick the least operationally heavy option that meets requirements. Typical progression:
1. Managed platforms (Vercel for Next.js, Render/Railway/Fly, AWS App Runner, Elastic Beanstalk) + managed DB (Atlas, RDS, Neon, Supabase).
2. Containers on ECS Fargate behind an ALB when you need more control, VPC networking, workers, or AWS-native integration.
3. Kubernetes (EKS) only with multiple teams/services and people to operate it.
Never introduce Terraform/K8s/service meshes in a feature task. Infrastructure as Code (Terraform/CDK) is recommended once there is more than one environment on AWS — propose it, don't force it.

### R2. CI on every PR
Pipeline order (fail fast, cheap first): checkout → setup Node (version from `.nvmrc`/`engines`) with package-manager cache → frozen-lockfile install → lint → typecheck → unit tests → integration tests (services via `services:` containers) → build → (optional) Docker build + scan → (optional) bundle-size check.
- Use path filters / affected-only in monorepos (Turborepo/Nx).
- `concurrency` to cancel superseded runs on the same branch.
- Required status checks + branch protection on `main`.

### R3. CD
- Build once, deploy the **same artifact** (image tag = git SHA) to staging, then production. Don't rebuild per environment (exception: frontends with build-time public vars — then build per env and say so).
- Staging deploy on merge to `main`; production via tag/release or manual approval (`environment: production` with required reviewers).
- Deployment strategy: rolling (default on ECS/App Runner) with health checks; blue/green or canary only when downtime/risk justify it.

### R4. Environments
- `development` (local), `staging` (prod-like, separate DB & secrets, safe test data), `production`. Optional preview environments per PR for frontends.
- Same config **keys** everywhere; different values. Validated at boot (`engineering-standards` R8).
- Never point staging at the production DB. Never copy production PII into staging without anonymization.

### R5. Secrets
- CI: GitHub Actions secrets scoped to environments; AWS access via **OIDC role assumption** (`aws-actions/configure-aws-credentials` with `role-to-assume`) — no long-lived AWS keys in GitHub.
- Runtime: AWS Secrets Manager / SSM Parameter Store (or platform env vars) injected into tasks; least-privilege IAM.
- Never echo secrets in logs; mask any derived values.

### R6. Database migrations in deploys
- Run migrations as a dedicated step before (or as part of) the deploy, once, with a lock (`prisma migrate deploy`, TypeORM `migration:run`).
- Migrations must be backward compatible with the currently running version (expand/contract — see `database` R7) so rolling deploys and rollbacks are safe.
- Destructive migrations happen in a later release, after confirming no code uses the old schema.

### R7. Rollback
- Keep previous images/releases; rollback = redeploy previous SHA (document the command/button).
- App rollback must not require DB rollback (thanks to expand/contract). Down-migrations are optional; don't rely on them for prod.
- Use feature flags for risky features to disable without redeploy.

### R8. Health checks
Liveness `/health` (process OK) and readiness `/health/ready` (DB/critical deps OK) wired to the load balancer/orchestrator. Readiness failures stop traffic; liveness failures restart. Graceful shutdown drains in-flight requests on SIGTERM.

### R9. Logs, metrics, errors, alerts
- Structured JSON logs to stdout; the platform ships them (CloudWatch Logs, Datadog, Grafana Loki). Include requestId, tenantId, userId (IDs only), route, status, duration.
- Error tracking: Sentry (or equivalent) for frontend + backend, with release = git SHA and source maps uploaded (not publicly served).
- Metrics to watch: request rate, error rate (5xx), latency p95/p99, CPU/memory, DB connections, queue depth (if queues).
- Alerts on symptoms users feel (error rate, latency, health check failures), routed to a channel someone watches. Avoid noisy alerts.
- Uptime check on the public URL.

### R10. Basic AWS mapping (when on AWS)
| Need | Service |
|---|---|
| Static SPA / assets | S3 (private) + CloudFront (OAC), cache-busting via hashed filenames |
| Containers | App Runner (simplest) or ECS Fargate + ALB |
| Next.js SSR | Vercel/Amplify Hosting, or container on ECS/App Runner (standalone output) |
| Postgres | RDS / Aurora (Multi-AZ for prod), automated backups + PITR |
| MongoDB | MongoDB Atlas (VPC peering/PrivateLink) |
| Secrets | Secrets Manager / SSM Parameter Store |
| Files | S3 private + presigned URLs |
| Jobs/queues | SQS + worker service, or BullMQ on ElastiCache Redis if already using Redis |
| Email | SES |
| Images registry | ECR (scan on push) |
| Logs/metrics | CloudWatch |
Place DBs in private subnets; only the load balancer is public. Security groups: least privilege.

### R11. Dependency & security automation
Dependabot/Renovate for updates (grouped, scheduled), `npm audit`/Trivy in CI, secret scanning + push protection on the repo.

## Implementation Guidelines

1. Inspect existing workflows, hosting target and environments.
2. Extend the existing pipeline rather than replacing it.
3. Add/modify jobs following R2/R3; use environments + OIDC for deploy credentials.
4. Ensure migrations, health checks and rollback path are defined.
5. Validate workflow syntax (`actionlint` if available) and describe what triggers what.

## Examples

**CI workflow (pnpm, Postgres service)**
```yaml
name: CI
on:
  pull_request:
  push: { branches: [main] }
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }
jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_USER: test, POSTGRES_PASSWORD: test, POSTGRES_DB: test }
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U test" --health-interval 5s --health-retries 10
    env:
      DATABASE_URL: postgres://test:test@localhost:5432/test
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm prisma migrate deploy
      - run: pnpm test -- --run
      - run: pnpm build
```

**Deploy job with OIDC (sketch)**
```yaml
  deploy-staging:
    needs: verify
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}, aws-region: ${{ vars.AWS_REGION }} }
      - uses: aws-actions/amazon-ecr-login@v2
      - run: |
          docker build -t $ECR_REPO:${{ github.sha }} .
          docker push $ECR_REPO:${{ github.sha }}
      # run migrations as one-off task, then update service to the new image tag
```

## Anti-patterns

- Deploying from a developer laptop; no CI checks before merge.
- Long-lived AWS access keys in GitHub secrets.
- Rebuilding a different artifact for prod than the one tested in staging.
- Migrations that break the currently running version.
- Staging sharing the production database.
- Logs as unstructured strings; no request IDs; no error tracking.
- Kubernetes/Terraform modules/service mesh for a single small app.
- Alerts nobody receives, or alerts on every warning.

## Checklist

- [ ] CI runs install (frozen), lint, typecheck, tests, build on PRs; required on main.
- [ ] Same artifact promoted staging → prod (or per-env build justified).
- [ ] Environments isolated (DB, secrets); config keys consistent and validated.
- [ ] Secrets via environment secrets / secret manager; AWS via OIDC.
- [ ] Migrations run once per deploy and are backward compatible.
- [ ] Rollback path documented and doesn't need DB rollback.
- [ ] Health/readiness checks wired; graceful shutdown.
- [ ] Structured logs, error tracking, key alerts in place.
- [ ] Infrastructure complexity justified by requirements.
