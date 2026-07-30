---
sidebar_position: 14
title: Accounts & Workspaces
---

# Accounts & Workspaces

## Accounts

You sign in with an email and password. The login page also handles
registration — creating an account automatically creates your personal
workspace. Sessions use short-lived access tokens refreshed transparently in
the background, so you stay signed in for up to 30 days per device.

The **Sessions** page lists your active sessions (device, IP address, created
and last-used times) and lets you revoke any of them remotely — signing out
elsewhere with one click. Logging out revokes the current session server-side.

Your display name, password, and sessions are all managed under
**[Settings → Account](/docs/user-guide/settings#account)**.

## Workspaces

A workspace scopes everything you create — agents, model configs and keys,
knowledge bases, routers, MCP servers, conversations, and usage. Nothing leaks
between workspaces: every request runs against exactly one workspace, and
members of one workspace can't see another's data.

- The **workspace switcher** in the sidebar shows your current workspace and
  lets you jump between the workspaces you belong to.
- **Create a workspace** from the switcher (or `/workspaces/new`) with just a
  name — you become its owner.
- **Workspace settings** lets you rename the active workspace.

Registering gives you a personal workspace automatically, so teams can start
without any setup.

### Members and roles

The **Members** page lists everyone in the workspace with their email, role, and
join date. Owners and admins can change a member's role or remove them.

| Role | What it means today |
| --- | --- |
| **Owner** | Full control. You get this on any workspace you create. |
| **Admin** | Can invite, remove, and re-role members. |
| **Builder** | Membership without member-management rights. |
| **Viewer** | Membership without member-management rights. |

Two guardrails always apply: you can't change or remove **your own**
membership, and the **last owner** can't be demoted or removed — a workspace
never ends up with nobody in charge.

:::note Where roles bite (and where they don't)
Roles are enforced on member and invitation management. They are **not yet
enforced on the rest of the platform**: any member can still create and edit
agents, keys, data sources, and other resources. Treat `builder` and `viewer` as
membership bookkeeping until per-resource permissions land, and don't rely on them
to keep someone out of a workspace's data.
:::

### Inviting teammates

From **Invite members**, enter one or more email addresses (up to 50) and pick the
role they should get. Owners and admins can invite.

Each invite produces a **link** to send to the invitee. The platform doesn't send
the email for you — copy the link and deliver it however your team communicates.
Invites expire after 7 days by default.

The result is reported per address, so a partial batch is never a failure: an
address that already belongs to the workspace is reported as such and nothing is
created, and re-inviting a pending address refreshes its link, role, and expiry.

Accepting works two ways:

- **New to the platform** — the invitee opens the link, sets a display name and
  password, and lands in the workspace already signed in. The account uses the
  invited email.
- **Already has an account** — the invitee is added to the workspace, but the link
  never signs them in; they sign in with their own password. They can also just
  accept the invite in-app: pending invites addressed to your email show up for
  you to accept directly.

Pending invites are listed for owners and admins, and can be **revoked**, which
makes the link stop working.

Workspace deletion is deliberately deferred until cascading cleanup is designed.

## API

See [Auth & Sessions](/docs/api-reference/auth) and
[Workspaces](/docs/api-reference/workspaces) in the API reference — including
the `X-Workspace-Id` header that selects the active workspace per request, the
[member endpoints](/docs/api-reference/workspaces#members), the
[invitation endpoints](/docs/api-reference/workspaces#invitations), and
[`POST /v1/auth/accept-invite`](/docs/api-reference/auth#post-v1authaccept-invite).
