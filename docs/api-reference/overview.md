---
sidebar_position: 1
title: Overview
---

# API Overview

REST API for creating, configuring, and chatting with LLM agents, built on
FastAPI. All application endpoints are mounted under the `/v1` prefix; health
probes live at the root.

Every endpoint page in this reference includes a runnable `curl` example against
a local server.

## Base URL

For local development:

```text
http://localhost:8000
```

All feature endpoints are under `/v1` (e.g. `http://localhost:8000/v1/agents`).
The interactive OpenAPI docs are served at `http://localhost:8000/docs` and the
raw schema at `http://localhost:8000/openapi.json`.

## Authentication

The API uses **JWT bearer authentication**. Register or log in through the
[auth endpoints](/docs/api-reference/auth) to obtain an access/refresh token
pair, then send the access token on every request:

```bash
TOKEN=$(curl -s -X POST http://localhost:8000/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email": "you@example.com", "password": "your-password"}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/v1/agents
```

Every example in this reference assumes `$TOKEN` holds a valid access token.
Access tokens expire after 30 minutes; use the refresh token to mint new ones.

The active **workspace** is selected per request with the optional
`X-Workspace-Id` header (it must name a workspace you are a member of); without
it, your earliest workspace membership is used. See
[Workspaces](/docs/api-reference/workspaces).

Health probes (`/health`, `/ready`) and the auth flows themselves
(`/v1/auth/register`, `/v1/auth/login`, `/v1/auth/refresh`, `/v1/auth/logout`)
are unauthenticated. Everything else returns `401` with the standard envelope
when the token is missing, invalid, or expired:

```json
{
  "error": {
    "code": "unauthorized",
    "message": "Missing credentials",
    "details": null
  }
}
```

:::note Legacy Basic auth
A shared-credential HTTP Basic mode (`ADMIN_USERNAME` / `ADMIN_PASSWORD`,
defaults `admin` / `change-me`) survives as a temporary escape hatch, **disabled
by default** — enable it with `ENABLE_LEGACY_BASIC_AUTH=true`. The legacy admin
is bound to the default workspace and cannot create workspaces.
:::

## Error envelope

Every error — validation failures, not-found, auth, runtime provider errors, and
unexpected server errors — is returned in one consistent shape:

```json
{
  "error": {
    "code": "not_found",
    "message": "Agent 'agt_missing' not found.",
    "details": null
  }
}
```

| Field | Type | Description |
| --- | --- | --- |
| `code` | string | Stable, machine-readable error code. See [Error Codes](/docs/api-reference/errors). |
| `message` | string | Human-readable description. |
| `details` | any \| null | Optional extra context. For `validation_error` this is a list of per-field validation problems. |

The most common codes are `validation_error` (422), `not_found` (404),
`unauthorized` (401), and `internal_error` (500). Errors raised during a chat
turn use provider-specific codes such as `provider_rate_limited` (429) or
`agent_timeout` (504) — see the [full error-code table](/docs/api-reference/errors).

## Pagination

List endpoints accept two query parameters and return a page envelope.

**Query parameters**

| Name | Type | Required | Default | Constraints | Description |
| --- | --- | --- | --- | --- | --- |
| `limit` | integer | No | `20` | `1 ≤ x ≤ 100` | Maximum number of items to return. |
| `offset` | integer | No | `0` | `x ≥ 0` | Number of items to skip. |

**Page response envelope**

| Field | Type | Description |
| --- | --- | --- |
| `items` | array | The page of results. Element type depends on the endpoint. |
| `total` | integer | Total number of items across all pages. |
| `limit` | integer | The `limit` that was applied. |
| `offset` | integer | The `offset` that was applied. |

```json
{
  "items": [],
  "total": 42,
  "limit": 20,
  "offset": 0
}
```

:::note
Some endpoints return **plain arrays** instead of page envelopes — for example
`GET /v1/patterns` and `GET /v1/agents/templates`. Each endpoint page states
which shape it returns.
:::

## Content types

Requests with a body use `application/json`. The non-streaming chat turn returns
JSON; the streaming chat turn returns `text/event-stream` (Server-Sent Events).

## Health

Unauthenticated liveness and readiness probes, mounted at the root (not under
`/v1`).

### `GET /health`

Liveness probe. Always returns `200` if the process is up.

```bash
curl http://localhost:8000/health
```

```json
{ "status": "ok" }
```

### `GET /ready`

Readiness probe. Verifies database connectivity (`SELECT 1`). Returns `200` when
ready, or `503` with code `not_ready` if the database is unreachable.

```bash
curl http://localhost:8000/ready
```

```json
{ "status": "ready" }
```
