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
`owner` membership.

## Roles

| Role | Meaning |
| --- | --- |
| `owner` | Full control. Created for whoever creates the workspace. |
| `admin` | Manages members and invites. |
| `builder` | Intended for creating and editing resources. |
| `viewer` | Intended for read-only access. |

Roles are **enforced on the member and invite endpoints below** — only `owner`
and `admin` may invite, remove, re-role, or view pending invites. Elsewhere on
the platform they are stored but not yet checked: any member of a workspace can
still read and write its agents, keys, and other resources. Treat non-owner roles
as membership bookkeeping until per-resource enforcement lands.

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

## Members

### `GET /v1/workspaces/{workspace_id}/members`

List the workspace's members, oldest first. Any member may call this. Returns a
**plain array**.

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Membership id — this is what member routes take, **not** the user id. |
| `user_id` | string | The member's user id. |
| `email` | string | The member's email. |
| `display_name` | string | The member's display name. |
| `role` | string | `owner`, `admin`, `builder`, or `viewer`. |
| `created_at` | string | When they joined (ISO-8601). |

### `PATCH /v1/workspaces/{workspace_id}/members/{member_id}/role`

Change a member's role. `owner`/`admin` only.

**Request body:** `{ "role": "builder" }` — one of `admin`, `builder`, `viewer`
(`owner` cannot be assigned through this route).

**Response** `200` — the updated member object. `403` when you are not permitted,
when you target **your own** membership, or when the member is the **last
owner**.

### `DELETE /v1/workspaces/{workspace_id}/members/{member_id}`

Remove a member. `owner`/`admin` only. **Response** `204`. `403` when you target
yourself or the last owner; `404` for an unknown member.

## Invitations

Invites are addressed to an email, carry a role, and expire (default 7 days,
`INVITE_TTL_SECONDS`). An invite becomes a membership in one of two ways: the
invitee opens the emailed link and calls
[`POST /v1/auth/accept-invite`](/docs/api-reference/auth#post-v1authaccept-invite)
with its token, or — if they already have an account — they accept it in-app from
`GET /v1/workspaces/invites/mine`.

### `POST /v1/workspaces/{workspace_id}/members/invite`

Invite one or more emails. `owner`/`admin` only.

**Request body**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `emails` | array\<string\> | Yes | 1–50 valid emails; duplicates within the request are collapsed. |
| `role` | string | Yes | `admin`, `builder`, or `viewer`. |

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/workspaces/ws_abc123/members/invite \
  -H 'Content-Type: application/json' \
  -d '{"emails": ["teammate@example.com"], "role": "builder"}'
```

**Response** `200` — one result per email, so a partial batch is never an error:

```json
{
  "results": [
    {
      "email": "teammate@example.com",
      "status": "success",
      "invite_url": "http://localhost:5173/invite/accept?token=…"
    }
  ]
}
```

| `status` | Meaning |
| --- | --- |
| `success` | Invite created (or an existing pending invite refreshed with a new token, role, and expiry). `invite_url` is the link to send. |
| `already_member` | That email already belongs to the workspace; nothing was created. |
| `failed` | The invite could not be created. |

The link's base comes from `FRONTEND_BASE_URL`. The platform does not send the
email itself — deliver `invite_url` however you like.

### `GET /v1/workspaces/{workspace_id}/invites`

Pending invites for the workspace, newest first. `owner`/`admin` only. Returns a
**plain array** of `{ id, email, role, status, expires_at, created_at }`.

### `DELETE /v1/workspaces/{workspace_id}/invites/{invite_id}`

Revoke an invite (its `status` becomes `revoked`, so the link stops working).
`owner`/`admin` only. **Response** `204`.

### `GET /v1/workspaces/invites/mine`

Pending, unexpired invites addressed to **your own** email, across all
workspaces — the in-app alternative to the emailed link for invitees who already
have an account. Returns a **plain array**.

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Invite id — pass it to the accept route below. |
| `workspace_id` / `workspace_name` | string | The workspace you were invited to. |
| `role` | string | The role you'd receive. |
| `invited_by_email` | string | Who invited you. |
| `created_at` / `expires_at` | string | ISO-8601 timestamps. |

### `POST /v1/workspaces/invites/{invite_id}/accept`

Accept an invite addressed to your email. No token needed — you are already
authenticated as that email.

**Response** `200` — the workspace object you just joined. `404` if the invite
isn't yours, was already used, or expired. Accepting when you are somehow already
a member marks the invite accepted without duplicating the membership.

## Not yet available

**DELETE is deliberately deferred** — cascading deletion of a workspace's
resources needs its own design.
