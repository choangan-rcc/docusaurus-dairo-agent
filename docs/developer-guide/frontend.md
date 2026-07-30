---
sidebar_position: 2
title: Frontend Architecture
---

# Frontend Architecture

The frontend is a React 18 + TypeScript application built with Vite, styled with
Tailwind CSS v4, and using TanStack Query for server state. It lives in
`frontend/`, and `@` aliases `frontend/src`.

## Layout

```
frontend/src/
  api/                  # shared HTTP client + cross-cutting types
  app/                  # App routes, Layout chrome, nav data module
  i18n/                 # i18next setup + locales/<lang>/<namespace>.json
  features/<feature>/   # one folder per backend surface
```

- **`src/api/`** — the shared HTTP client: attaches the JWT bearer token and the
  active `X-Workspace-Id` header, transparently refreshes an expired access token,
  emits an `api-unauthorized` event when auth is unrecoverable, and unwraps the
  platform's error envelope into a typed `ApiError`.
- **`src/app/`** — routing and chrome. `nav.ts` is a **pure data module** (nav
  groups, icons, labels as i18n keys) so it can be unit-tested without i18n or
  router setup.
- **`src/features/<feature>/`** — one folder per backend surface: `agents`,
  `playground`, `patterns`, `knowledge-bases`, `data-sources`, `mcp-servers`,
  `model-configs`, `model-routers`, `guardrails`, `observability`, `usage`,
  `memories`, `workspaces`, `settings`, and `auth`. Each feature folder owns its
  own `api.ts`, `types.ts`, pages, and components.

## Internationalization

`src/i18n/` configures i18next with **statically bundled** JSON resources — one
namespace per feature area (`common`, `nav`, `auth`, `agents`), one folder per
locale under `locales/`. Nine languages ship today (`en`, `vi`, `ja`, `ko`, `zh`,
`hi`, `ms`, `es`, `pt-BR`), with `en` as the fallback. The choice persists in
`localStorage["lang"]`, the same idiom as the theme.

Adding a language is a new folder under `locales/` plus an entry in `LANGUAGES`;
adding a namespace is a new JSON in **every** locale plus a key in `resources`.

## Pure logic is plain TypeScript

Anything that can be expressed without the DOM — form state, SSE transcript
reduction, chart specs, upload state — is split into plain `.ts` modules with
colocated `*.test.ts` vitest tests that run in a node environment (no DOM, no
rendering). This keeps the most intricate logic in the codebase cheap to test.

```bash
npx vitest run src/features/playground/transcript.test.ts   # single test file
```

## Development

```bash
cd frontend
npm install
npm run dev          # Vite on port 5173
```

The dev server proxies `/v1`, `/health`, and `/ready` to the backend at
`127.0.0.1:8000`, so run the backend first.

Other scripts:

```bash
npm run build        # tsc --noEmit && vite build
npm run typecheck    # TypeScript only
npm run lint         # eslint src
npm run test         # vitest run
```

In CI, the `frontend` required check runs lint, typecheck, tests, and a
production build — all four must pass before a PR can merge.

## Production image

The production frontend is served by nginx (see `frontend/Dockerfile` and
`frontend/nginx.conf`). In Kubernetes, the Ingress routes `/v1` and `/health` to
the backend and everything else to the frontend — see
[Deployment](/docs/developer-guide/deployment).
