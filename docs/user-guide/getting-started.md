---
sidebar_position: 2
title: Getting Started
---

# Getting Started

This page walks through running DAIRO AI Factory locally, with the platform's
zero-config defaults, and then optionally pointing it at a real LLM provider and a
fuller local infrastructure stack.

## Prerequisites

- [`uv`](https://docs.astral.sh/uv/) — used to install and run the Python backend
- Python 3.11 or newer
- Node.js 18 or newer — used to run the frontend

## Run the backend

From the repository root:

```bash
uv sync
make dev
```

`make dev` starts the backend with `uvicorn` on port 8000, with automatic reload on
code changes.

Out of the box, no additional configuration is required. The backend runs with the
following zero-config defaults:

- SQLite as the database
- `fake` as the LLM provider — no API key needed
- An in-process ingestion queue
- Local file storage
- Chroma as the vector store

Interactive API documentation is available at `http://127.0.0.1:8000/docs`.

### Signing in

The platform uses email/password accounts with JWT sessions. On a fresh
install, open the frontend and use **Create your account** on the login page —
registering also creates your personal workspace. Alternatively, seed a first
admin user at boot by setting `BOOTSTRAP_ADMIN_EMAIL`,
`BOOTSTRAP_ADMIN_PASSWORD`, and `BOOTSTRAP_ADMIN_DISPLAY_NAME` in `.env`.

A legacy shared-credential HTTP Basic mode (`ADMIN_USERNAME` /
`ADMIN_PASSWORD`, defaults `admin` / `change-me`) still exists for transition
purposes but is **disabled by default**; enable it with
`ENABLE_LEGACY_BASIC_AUTH=true` if you need it.

## Run the frontend

In a separate terminal:

```bash
cd frontend && npm install && npm run dev
```

This starts the frontend with Vite on port 5173. The dev server proxies `/v1`,
`/health`, and `/ready` requests to the backend at `127.0.0.1:8000`, so the backend
must be running for the frontend to work.

## Using a real LLM provider

The `fake` provider is useful for trying the platform out or running it fully
offline, but it doesn't call a real model. To have agents use Anthropic's Claude
models instead, set the following in your `.env` file:

```bash
LLM_PROVIDER=anthropic
ANTHROPIC_API_KEY=...
```

## Optional: fuller local infrastructure

The zero-config defaults (SQLite, local storage, Chroma) are enough to explore the
platform, but you can also run it against a more production-like local stack —
Postgres with pgvector, Redis, MinIO, and Phoenix — using Docker Compose:

```bash
make infra-up
```

After the infrastructure is up, point the backend at Postgres by setting the
following in your `.env`:

```bash
DATABASE_URL=postgresql+asyncpg://agentic:agentic@localhost:5432/agentic
```

Then apply the database migrations:

```bash
uv run alembic upgrade head
```

## Optional: features that need extra setup

Two features are off (or unusable) with the bare defaults.

### Data sources

Connecting a [data source](/docs/user-guide/data-sources) stores encrypted
credentials, so a master key is required:

```bash
# Generate one:
#   python -c "import os,base64;print(base64.b64encode(os.urandom(32)).decode())"
CREDENTIAL_ENCRYPTION_KEY=<base64 32 bytes>
```

The connectors are launched with `uvx`, which downloads the pinned connector
package on first use — so the machine running the backend needs network access to
the Python package index, plus reachability to the database you're connecting.

### Long-term memory

[Memory](/docs/user-guide/memory) is off by default, and **learning** additionally
requires the arq worker (which needs Redis, from `make infra-up`):

```bash
MEMORY_ENABLED=true
INGEST_QUEUE=arq
REDIS_URL=redis://localhost:6379
```

Then run the worker alongside the API in a third terminal:

```bash
make worker
```

Without the worker, reading memory still works but nothing new is ever learned —
**Settings → Memory & privacy** will say so rather than pretending otherwise.

## Next steps

With the backend and frontend running, the next step is to create your first agent
and try it out. The Agents page in this guide covers creating agents from templates
and configuring them, and the Playground page covers testing an agent with
streaming chat.
