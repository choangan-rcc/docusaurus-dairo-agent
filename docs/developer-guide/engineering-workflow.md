---
sidebar_position: 3
title: Engineering Workflow
---

# Engineering Workflow

These are **binding conventions**, not suggestions. Git refs drive deployments and
database migrations run against real environments, so mistakes in this workflow
reach production. When in doubt, follow the rule and ask.

## Environments

A commit's Git ref decides where it deploys. Images are built **once per commit
SHA** and promoted through environments — never rebuilt. What you tested is
exactly what ships.

| Git event | Deploys to | Approval |
| --- | --- | --- |
| push to `dev` | `development` | none |
| push to `main` | `staging` | none |
| tag `v*.*.*` | `production` | required review |

Each GitHub Environment holds its own secrets (including `DATABASE_URL`), so the
pipeline migrates the correct database per environment.

## Branching

The repo uses environment branches (a simple GitLab Flow):

```
feature/*  →  dev (development)  →  main (staging)  →  tag v*.*.* (production)
```

- Never push directly to `main` or `dev` — all changes land via Pull Request.
- Branch off `dev` for new work; merge features into `dev` first. `main` only
  ever receives merges from `dev`.
- Name branches by intent: `feature/<desc>`, `fix/<desc>`, or `hotfix/<desc>`
  (hotfixes branch from `main`).
- Always `git pull` the target branch before branching off it.

## Pull requests

Branch protection on `main` and `dev` blocks a merge unless:

- at least **1 approving review**,
- all required status checks pass — `backend` (ruff + pytest), `frontend`
  (lint + typecheck + test + build), and `alembic-heads` (fails if more than one
  migration head exists),
- the branch is up to date with its target, and
- all review conversations are resolved.

Keep PRs small and focused — one logical change per PR — and rebase on the
latest `dev` before requesting review:

```bash
git fetch origin && git rebase origin/dev
```

## Commit messages

Use **Conventional Commits**: `<type>: <short summary>` with types `feat`,
`fix`, `chore`, `refactor`, `docs`, `test`, `ci`. Database-history commits use
`chore(db): ...` (e.g. `chore(db): merge alembic heads`).

## Releases

A release is a deliberate, named version of `main` shipped to production:

```bash
git checkout main && git pull origin main
git tag v1.2.0
git push origin v1.2.0
```

Pushing the tag triggers CD; the production deploy **pauses for a required
reviewer** before any migration runs. Versioning follows SemVer
(`vMAJOR.MINOR.PATCH`): MAJOR for breaking changes, MINOR for
backward-compatible features, PATCH for bug fixes.

Two hard rules:

- Only tag commits that are on `main` and have passed staging.
- Never move or delete a published tag — to fix a bad release, ship a new,
  higher version.

## Secrets and configuration

- Never commit secrets. `.env` is git-ignored; `.env.example` documents every key.
- Per-environment config lives in GitHub Environment secrets; each environment
  defines `DATABASE_URL` in async-driver form (`postgresql+asyncpg://...`).
- `CREDENTIAL_ENCRYPTION_KEY` and `ADMIN_PASSWORD` are required and must be
  unique per environment in staging and production.
