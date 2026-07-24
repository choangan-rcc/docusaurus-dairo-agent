---
sidebar_position: 12
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

### Roles and invitations

Member roles (`owner` / `admin` / `builder` / `viewer`) are part of the data
model but not yet enforced — today every member is an owner. The invite-members
page exists in the UI but the invitation API isn't wired up yet; inviting
teammates is coming in an upcoming release. Workspace deletion is likewise
deliberately deferred until cascading cleanup is designed.

## API

See [Auth & Sessions](/docs/api-reference/auth) and
[Workspaces](/docs/api-reference/workspaces) in the API reference — including
the `X-Workspace-Id` header that selects the active workspace per request.
