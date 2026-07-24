---
sidebar_position: 5
title: Deployment & CI/CD
---

# Deployment & CI/CD

How a commit becomes a running release, and how the platform is deployed to
Kubernetes: the GitHub Actions pipelines, the two Helm charts, and the one-time
setup each environment needs.

## Overview

```text
push to dev  ──▶ CI (lint/test) ──▶ build image sha-<sha> ──▶ CD ──▶ development
push to main ──▶ CI (lint/test) ──▶ build image sha-<sha> ──▶ CD ──▶ staging
tag  v*.*.*  ─────────────────────────────(no rebuild)─────▶ CD ──▶ production
```

**Build once, promote by reference.** Images are built once per commit and tagged
with the immutable git SHA (`sha-<short-sha>`). Deploys — including production —
always reference that SHA tag. A production release is a git tag pointing at a
commit whose image already exists and already ran in staging; nothing is ever
rebuilt per environment.

Two separately deployed Helm charts:

| Chart | What | Lifecycle |
| --- | --- | --- |
| `charts/agentic-platform` | Backend API, frontend, migration Job, Ingress | Every merge/tag, automated via `cd.yml`, `--atomic` |
| `charts/agentic-infra` | Postgres+pgvector, Redis, MinIO, Phoenix | Rare, installed manually per environment |

They are split on purpose: the app chart rolls (and auto-rolls-back) on every
deploy; datastores must never do that.

## CI

`ci-backend.yml` and `ci-frontend.yml` trigger on PRs and pushes to `dev`/`main`,
path-filtered so backend-only changes don't run frontend jobs and vice versa.

- **Backend:** `ruff check` + `ruff format --check`, then `pytest` (in-memory
  SQLite + fake LLM provider — no services, no network).
- **Frontend** (in `frontend/`): `eslint`, `tsc --noEmit`, `vitest`, `vite build`.

**Image build** (push events only, never PRs) pushes to GHCR:

```text
ghcr.io/san-data-systems/dairo-ai-orchestrator-backend:sha-<sha>   (+ branch tag)
ghcr.io/san-data-systems/dairo-ai-orchestrator-frontend:sha-<sha>  (+ branch tag)
```

Auth uses the workflow's built-in `GITHUB_TOKEN`. Trivy scans each image
(HIGH/CRITICAL) and uploads SARIF to the repo Security tab — currently
report-only. Dependabot watches uv, npm, both Dockerfiles, and the workflows,
weekly.

## CD

One workflow (`cd.yml`), three destinations, resolved from the git ref:

| Ref | GitHub Environment | Namespace |
| --- | --- | --- |
| `dev` branch | `development` | `agentic-platform-development` |
| `main` branch | `staging` | `agentic-platform-staging` |
| tag `v*.*.*` | `production` | `agentic-platform-production` |

The deploy job runs under the matching **GitHub Environment**, which supplies
that environment's secrets — a development kubeconfig can never touch production.
Production should additionally have a **required reviewer** configured; the
deploy then pauses for human approval.

Each deploy is a single `helm upgrade --install` with:

- `-f values-<environment>.yaml` — per-env config (replicas, hostname, LLM provider)
- `--set backend.image.tag=sha-<sha>` / `frontend.image.tag=sha-<sha>`
- `--set secrets.*` — injected from the GitHub Environment's secrets
- `--atomic --wait --timeout 5m` — if new pods never go Ready, Helm rolls
  everything back to the previous release automatically.

### Migrations during deploy

`alembic upgrade heads` runs as a Helm **pre-upgrade hook Job** using the same
backend image being deployed. Helm blocks the rollout until the Job succeeds:

1. `helm upgrade` starts → migration Job runs against the new image.
2. Job fails → release aborts, **no pod ever rolls**.
3. Job succeeds → Deployments roll to the new SHA.

Migrations never run inside the app container or an initContainer — with
multiple replicas that would race on DDL.

:::caution
`helm rollback` / `--atomic` does **not** reverse migrations. Write
expand/contract (backward-compatible) migrations so the previous image runs
against the new schema — see [Database Migrations](/docs/developer-guide/migrations).
:::

### Rollback

- **Automatic:** failed readiness within the 5-minute window → `--atomic` reverts.
- **Manual** (bad behavior that passes health checks):

  ```bash
  helm -n agentic-platform-<env> history agentic-platform
  helm -n agentic-platform-<env> rollback agentic-platform <revision>
  ```

## App chart — `charts/agentic-platform/`

Backend (FastAPI) and frontend (nginx) Deployments + ClusterIP Services, one
Ingress, migration Job, ConfigMap (non-secret env), Secret (populated at deploy
time — real values never in git), PodDisruptionBudget, and a default-deny
NetworkPolicy that admits only the ingress controller.

- Backend: rolling update `maxUnavailable: 0 / maxSurge: 1`, liveness `/health`,
  readiness `/ready` (gates traffic on the DB), non-root uid 10001, read-only
  root filesystem.
- Ingress routes `/v1` and `/health` to the backend, everything else to the
  frontend. Hostnames are placeholders in `values-*.yaml` — set real ones.
- Per-env overlays: `values-development.yaml` (1 replica, fake LLM),
  `values-staging.yaml` (2 replicas, Anthropic + Phoenix),
  `values-production.yaml` (3 replicas, TLS enabled).

## Infra chart — `charts/agentic-infra/`

Stateful backing services, all ClusterIP-only, nothing exposed via Ingress:

| Component | Kind | Notes |
| --- | --- | --- |
| Postgres 16 + pgvector | StatefulSet + PVC | First-boot init enables `vector`, creates the `phoenix` database |
| Redis 7 | StatefulSet + PVC | `appendonly yes` — the arq queue survives restarts |
| MinIO | StatefulSet + PVC | S3 API only; a post-install Job creates the `agentic-kb` bucket |
| Phoenix | Deployment | Postgres-backed, `PHOENIX_ENABLE_AUTH=true`; UI via port-forward only |

Chroma is **deliberately absent** from the cluster: all server releases ≤ 1.5.9
carry an unpatched pre-auth RCE, and the app defaults to embedded Chroma anyway.
Do not deploy the Chroma server to a shared cluster until a patched image exists.

Install once per environment:

```bash
helm upgrade --install agentic-infra charts/agentic-infra \
  --namespace agentic-platform-staging --create-namespace \
  --set storageClass=<vsphere-csi-class> \
  --set networkPolicy.appNamespace=agentic-platform-staging \
  --set secrets.postgresPassword=<pw> \
  --set secrets.minioRootPassword=<pw> \
  --set secrets.phoenixSecret=<long-random-string>
```

In-cluster DNS names the app secrets should point at:

```text
DATABASE_URL                postgresql+asyncpg://agentic:<pw>@agentic-infra-postgres:5432/agentic
REDIS_URL                   redis://agentic-infra-redis:6379
S3_ENDPOINT                 http://agentic-infra-minio:9000     (S3_BUCKET=agentic-kb)
PHOENIX_COLLECTOR_ENDPOINT  http://agentic-infra-phoenix:6006
```

Phoenix UI: `kubectl -n <ns> port-forward svc/agentic-infra-phoenix 6006:6006`,
then create a system API key in the Phoenix UI (Settings → API Keys) and set
`PHOENIX_API_KEY` in the app environment's secrets.

## One-time environment setup

Per environment (`development`, `staging`, `production`):

1. **Cluster prerequisites:** ingress-nginx installed, a default StorageClass or
   an explicit vSphere CSI class, and DNS for the Ingress hostname pointing at
   the ingress NodePort/VIP.
2. **Namespace + infra chart:** run the `agentic-infra` install above.
3. **GitHub Environment:** create it (on `production`, add a required reviewer)
   and add these environment secrets:

   | Secret | Value |
   | --- | --- |
   | `KUBE_CONFIG` | base64-encoded kubeconfig, RBAC-scoped to this env's namespace only — not cluster-admin |
   | `DATABASE_URL` | see the DNS table above |
   | `ADMIN_USERNAME` / `ADMIN_PASSWORD` | app Basic-auth admin credential |
   | `ANTHROPIC_API_KEY` | per-environment key |

4. **Real hostname** in `charts/agentic-platform/values-<env>.yaml`.

## Releasing to production

```bash
git tag v1.2.3 <commit-on-main>   # a commit whose image already ran in staging
git push origin v1.2.3
# → cd.yml resolves environment=production → pauses for reviewer approval
# → migration Job → atomic rollout of the exact staging-tested image
```

## Known gaps (deliberate, MVP-stage)

- **No Postgres backups.** Single replica, no PITR. Required before real
  production data: pgBackRest or a `pg_dump` CronJob shipping to MinIO.
- **`KUBE_CONFIG` is a long-lived credential.** Mitigate by scoping each
  kubeconfig's ServiceAccount to its single namespace and rotating periodically.
- **Trivy is report-only.** Flip `exit-code` to `1` once the baseline is clean so
  HIGH/CRITICAL findings block the build.
- **Single-replica datastores.** No HA Postgres/Redis/MinIO. Acceptable for
  dev/staging; revisit before production traffic matters.
