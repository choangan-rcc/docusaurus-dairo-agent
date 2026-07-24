---
sidebar_position: 1
title: Backend Architecture
---

# Backend Architecture

The backend is a Python/FastAPI application that lives in `src/agentic_platform/` and
is managed with [`uv`](https://docs.astral.sh/uv/). This page describes how the
application is put together and the conventions that every subsystem follows.

## Application startup

The FastAPI app is created by an **app factory** in `main.py`. Its lifespan handler
runs the platform's startup sequence:

1. Set up **Phoenix tracing** (this must happen before any LlamaIndex engine is
   built, so instrumentation hooks are in place first).
2. Initialize the database.
3. Seed the default workspace.
4. Sync the **pattern catalog** from code into the database.
5. When the in-process ingestion queue is used, sweep stranded ingestion jobs left
   behind by a previous process.

Configuration is entirely env-driven via `config.py`, which defines a
pydantic-settings `Settings` class exposed through an `lru_cache`'d
`get_settings()`. `config.py` is treated as a fixed contract — its import order is
deliberately excluded from ruff's isort rule.

The platform runs with **zero-configuration defaults**: SQLite database, the `fake`
LLM provider (no API key), an in-process ingestion queue, local file storage, and
Chroma as the vector store. Every one of those defaults can be swapped for a
production-grade backend (Postgres + pgvector, arq/Redis, S3/MinIO, OpenSearch)
through environment variables alone.

## Key conventions

### Org-scoped repository layer

Feature code **never queries models directly**. All database access goes through the
repository layer in `database/repository.py`. A repository is constructed with an
`org_id`, and every query it issues filters on that `org_id`.

The platform currently runs in single-workspace mode against a seeded
`org_default` workspace, but every table already carries `org_id`. When
multi-tenancy lands, the only change needed is how the `OrgId` dependency in
`api/deps.py` resolves the organization from the authenticated principal — no
feature code changes.

### Single error envelope

Every error response — validation, not-found, auth, provider failures, unexpected
server errors — uses one consistent shape, produced by `api/errors.py`:

```json
{ "error": { "code": "not_found", "message": "...", "details": null } }
```

See the [API Reference](/docs/api-reference/overview) for the full error-code table.

### Authentication

All `/v1` routes require HTTP Basic admin auth; `/health` and `/ready` are
unauthenticated and mounted at the root.

### The registry idiom

LLM providers, agent patterns, tools, embeddings backends, vector databases, and
storage backends each **self-register** via a `@register_*` decorator. Adding an
implementation is one new module plus an import in the package's `__init__.py` —
no changes to existing implementations.

Two rules keep this safe:

- Class paths are never stored in the database.
- Nothing is ever imported dynamically from database data.

The database only ever stores stable string identifiers (`pattern_id`, provider
names, backend names) that are looked up in the in-code registries.

### Fake implementations for offline CI

The platform ships a `fake` LLM provider and `fake` embeddings backend. The entire
test suite must pass with **no network access and no API keys** — CI runs against
in-memory SQLite and the fake providers.

## Agent runtime (`agents/`) — hexagonal

The agent runtime follows a hexagonal (ports-and-adapters) layout:

- **`domain/`** — engine-agnostic entities (`AgentTurnResult`, `ToolTrace`) and
  ports: the `AgentEngine` interface, `EngineRequest`, and streamed `EngineEvent`s
  — including `ApprovalRequired`, which powers human-in-the-loop tool approval.
- **`application/`** — the `AgentService` use case. A chat turn flows through
  guardrails → engine → persistence → usage capture, all through org-scoped
  repositories.
- **`infrastructure/`** — adapters: the LlamaIndex engine, the multi-provider LLM
  registry (fake / anthropic / openai / gemini / bedrock), built-in tools, MCP tool
  adapters, and knowledge-base retrieval tools.
- **`patterns/`** — named, versioned recipes (`function_agent`, `react_agent`) that
  map a `pattern_id` to executable code. The `agent_patterns` database table is
  only the display/enable catalog — the code registry is the source of truth.

Every pattern config inherits `PatternSafetyConfig` (max iterations, timeout) with
`extra="ignore"`, so agents created against an older pattern version keep working
after a pattern upgrade — unknown config fields are dropped, not rejected.

:::note
Always import from `agentic_platform.agents` (the package), not its submodules —
the internal layout may evolve.
:::

## Other subsystems

| Package | Responsibility |
| --- | --- |
| `llm/` | Provider catalog, BYOK key verification, model resolution, and intelligent routing (model routers pick a model per request). |
| `ingestion/` | Knowledge-base document pipeline (parse → chunk → embed → index). Queue is in-process by default, or arq/Redis with `INGEST_QUEUE=arq`; under arq the worker owns the stale-job sweep. |
| `vectordb/` | Pluggable vector stores (Chroma, OpenSearch), selected via settings. |
| `embeddings/` | Pluggable embeddings backends (including `fake` for CI). |
| `storage/` | Pluggable blob storage (local filesystem, S3/MinIO). |
| `security/crypto.py` | AES-256-GCM encryption for stored BYOK keys (`CREDENTIAL_ENCRYPTION_KEY`). API responses only ever expose a masked `key_hint`. |
| `observability.py` | Arize Phoenix tracing, gated by `PHOENIX_ENABLED` (off by default). `api/observability.py` serves trace data to the frontend. |

## API surface

All feature routes are mounted under `/v1` and wired together in `api/router.py`:
`agents` (plus templates, duplicate, and the playground chat with SSE streaming),
`model-configs`, `model-routers`, `patterns`, `mcp-servers`, `knowledge-bases`
(plus per-agent attachment routers), `providers/{provider}/models`, `tools`, and
`observability`. Interactive OpenAPI docs are served at `/docs`.

## Tests

The suite uses `pytest` with `asyncio_mode = "auto"` and `pythonpath = ["src"]`.
`tests/conftest.py` gives every test:

- a fresh **in-memory SQLite** database with the pattern catalog pre-synced,
- the `fake` LLM provider force-patched in, and
- a test encryption key.

A developer's local `.env` must never leak into the suite. Because
`get_settings()` is lru-cached, tests patch attributes on the settings singleton
rather than re-instantiating it.

```bash
make test                                   # full suite
uv run pytest tests/test_chat.py -q         # single file
uv run pytest tests/test_chat.py -k name -q # single test
```
