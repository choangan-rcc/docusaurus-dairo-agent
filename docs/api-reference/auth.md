---
sidebar_position: 2
title: Auth & Sessions
---

# Auth & Sessions

Email/password authentication issuing **JWT bearer tokens** (HS256), with
server-side session records that make logout and remote revocation effective.

- **Access tokens** are short-lived (30 minutes, `JWT_ACCESS_TTL_SECONDS`) and
  stateless.
- **Refresh tokens** are long-lived (30 days, `JWT_REFRESH_TTL_SECONDS`) and tied
  to a server-side session row — refresh only succeeds while that session exists
  and is unrevoked.

The auth flows themselves (`register`, `login`, `refresh`, `logout`) are
unauthenticated; the session and profile endpoints require a bearer token. The
active workspace is **not** a token claim — it travels per request in the
`X-Workspace-Id` header.

## `POST /v1/auth/register`

Create an account. Also creates the user's personal workspace
(`"<display_name>'s workspace"`) with an `owner` membership.

**Request body**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | string | Yes | Valid email; lowercased. |
| `password` | string | Yes | 8–128 chars. |
| `display_name` | string | Yes | 1–255 chars. |

```bash
curl -X POST http://localhost:8000/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email": "you@example.com", "password": "a-strong-password", "display_name": "You"}'
```

**Response** `201`:

```json
{
  "access_token": "eyJ…",
  "refresh_token": "eyJ…",
  "user": { "id": "usr_1a2b3c4d", "email": "you@example.com", "display_name": "You" }
}
```

Duplicate email returns `409` with code `conflict`.

## `POST /v1/auth/accept-invite`

Redeem a [workspace invite](/docs/api-reference/workspaces#invitations) token.
Unauthenticated — this is the endpoint behind the emailed
`/invite/accept?token=…` link, and it handles both a brand-new invitee and one
who already has an account.

**Request body**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `token` | string | Yes | The invite token from the link. |
| `password` | string | Conditional | 8–128 chars. Required only when the invited email has **no** account yet. |
| `display_name` | string | Conditional | 1–255 chars. Same condition as `password`. |

Two outcomes, distinguished by `status`:

| `status` | When | Response |
| --- | --- | --- |
| `registered` | The invited email had no account. One is created (email taken from the invite, not the body) and joined to the workspace. | `{ status, workspace_id, tokens: { access_token, refresh_token, user } }` |
| `joined_existing_account` | The email already has an account. Membership is granted, but **no tokens are issued** — holding an invite link must never log anyone into an existing account. The invitee signs in with their own password. | `{ status, workspace_id, tokens: null }` |

```bash
curl -X POST http://localhost:8000/v1/auth/accept-invite \
  -H 'Content-Type: application/json' \
  -d '{"token": "…", "password": "a-strong-password", "display_name": "Teammate"}'
```

An invalid, expired, or already-used invite returns `404`. A new-account invite
missing `password`/`display_name` returns `422`. The invite is marked accepted on
success, so the link is single-use.

## `POST /v1/auth/login`

Exchange email/password for a token pair. Returns the same shape as register.
Unknown email, inactive user, and wrong password all return the **same** `401`
("Invalid email or password").

```bash
curl -X POST http://localhost:8000/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email": "you@example.com", "password": "a-strong-password"}'
```

## `POST /v1/auth/refresh`

Mint a new access token from a refresh token.

**Request body:** `{ "refresh_token": "eyJ…" }`

**Response** `200`: `{ "access_token": "eyJ…" }`

Returns `401` if the token is invalid or expired, its session was revoked, or
the user is gone/inactive.

## `POST /v1/auth/logout`

Revoke the session behind a refresh token.

**Request body:** `{ "refresh_token": "eyJ…" }`

**Response** `204`. Calling again with the same token returns `401` (already
revoked).

## `GET /v1/auth/sessions`

List the caller's active sessions (unrevoked, unexpired), newest first.
Requires a bearer token.

```json
{
  "sessions": [
    {
      "id": "ses_9f8e7d6c",
      "user_agent": "Mozilla/5.0 …",
      "ip_address": "203.0.113.7",
      "created_at": "2026-07-20T08:00:00Z",
      "last_used_at": "2026-07-24T09:15:00Z",
      "expires_at": "2026-08-19T08:00:00Z"
    }
  ]
}
```

## `DELETE /v1/auth/sessions/{session_id}`

Revoke one of your own sessions. `204` on success; a session that isn't yours
(or doesn't exist) returns `404` — cross-user revocation is indistinguishable
from not-found.

## `GET /v1/auth/me`

The caller's profile and workspace memberships.

```json
{
  "user": { "id": "usr_1a2b3c4d", "email": "you@example.com", "display_name": "You" },
  "memberships": [
    { "workspace_id": "ws_default", "name": "Default Workspace", "role": "owner" }
  ]
}
```

## `PATCH /v1/auth/me`

Update your own profile — display name only for now.

**Request body:** `{ "display_name": "New Name" }` (1–255 chars)

**Response** `200`: `{ "id": "usr_1a2b3c4d", "email": "you@example.com", "display_name": "New Name" }`

## `POST /v1/auth/change-password`

Change your password after re-verifying the current one. Requires a bearer token.

**Request body**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `current_password` | string | Yes | Must match your current password. |
| `new_password` | string | Yes | 8–128 chars. |

**Response** `204`. A wrong current password returns **`400
invalid_current_password`**, deliberately not `401` — clients treat any `401` as
an expired session and would bounce you to the login page.

Existing sessions are **not** revoked by a password change; revoke them from
`DELETE /v1/auth/sessions/{session_id}` if that's what you want.

## Configuration

| Env var | Default | Purpose |
| --- | --- | --- |
| `JWT_SECRET` | `dev-only-jwt-secret` | Token signing secret. The app **refuses to boot** outside a local environment with the default/empty secret. |
| `JWT_ACCESS_TTL_SECONDS` | `1800` | Access-token lifetime. |
| `JWT_REFRESH_TTL_SECONDS` | `2592000` | Refresh-token lifetime (30 days). |
| `BOOTSTRAP_ADMIN_EMAIL` / `BOOTSTRAP_ADMIN_PASSWORD` / `BOOTSTRAP_ADMIN_DISPLAY_NAME` | — | Seed a first real user into the default workspace at boot. |
| `INVITE_TTL_SECONDS` | `604800` | Workspace-invite lifetime (7 days). |
| `FRONTEND_BASE_URL` | `http://localhost:5173` | Base of the `invite_url` returned when inviting members. |
| `ENABLE_LEGACY_BASIC_AUTH` | `false` | Temporary escape hatch: accept shared-credential HTTP Basic (`ADMIN_USERNAME` / `ADMIN_PASSWORD`). The legacy admin is workspace-blind (bound to the default workspace). |
