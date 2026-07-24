---
sidebar_position: 3
title: Workspaces
---

# Workspaces

A **workspace** scopes every resource on the platform — agents, model configs,
knowledge bases, routers, usage, and so on. Every feature request runs against
exactly one workspace, selected by the `X-Workspace-Id` header (which must name
a workspace the caller is a member of, else `403`); without the header, the
caller's earliest membership is used.

Registering a user automatically creates their personal workspace with an
`owner` membership. Roles (`owner` / `admin` / `builder` / `viewer`) are stored
but not yet enforced — v1 writes `owner` for everyone.

## Workspace object (`WorkspaceOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Workspace id (prefix `ws_`). The seeded default is `ws_default`. |
| `name` | string | Workspace name. |
| `slug` | string \| null | Optional unique slug. |
| `organization_id` | string \| null | Owning organization, when set. |
| `settings` | object | Free-form settings JSON (default `{}`). |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

## `GET /v1/workspaces`

List the caller's workspaces (memberships, oldest first). Returns a plain
array. The legacy Basic-auth admin sees all workspaces.

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/v1/workspaces
```

## `POST /v1/workspaces`

Create a workspace; the creator becomes an `owner` member.

**Request body:** `{ "name": "My team" }` (1–255 chars)

**Response** `201` — the workspace object. The legacy Basic-auth admin cannot
create workspaces (`403`).

## `GET /v1/workspaces/{workspace_id}`

Fetch a workspace. `403` for non-members, `404` for unknown ids.

## `PATCH /v1/workspaces/{workspace_id}`

Update `name` and/or `settings` (PATCH semantics — only provided fields change).

```bash
curl -H "Authorization: Bearer $TOKEN" -X PATCH \
  http://localhost:8000/v1/workspaces/ws_abc123 \
  -H 'Content-Type: application/json' \
  -d '{"name": "Platform team"}'
```

## Not yet available

- **DELETE is deliberately deferred** — cascading deletion of a workspace's
  resources needs its own design.
- **Member/invitation endpoints do not exist yet**; the UI's invite page is a
  stub pending `POST /v1/workspaces/{id}/invitations`.
