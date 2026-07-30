---
sidebar_position: 15
title: Settings
---

# Settings

**Settings** collects the preferences and personal controls that aren't tied to a
single resource. Each tab is a real, linkable route under `/settings`.

## General

Appearance and language.

- **Theme** — Light, Dark, or System. System follows your operating system's
  preference and keeps following it if you change it later.
- **Language** — the interface language. Your choice persists across sessions.

The console is available in nine languages, each listed under its own name:

| | | |
| --- | --- | --- |
| English | Tiếng Việt | 日本語 |
| 한국어 | 简体中文 | हिन्दी |
| Bahasa Melayu | Español | Português (Brasil) |

:::note
This setting translates the **console interface**. What language an *agent*
replies in is a property of the agent (its `language` setting and system prompt),
and — if [long-term memory](/docs/user-guide/memory) is on — a language
preference you state in chat is one of the facts every agent in the workspace can
remember.
:::

## Account

Your profile and credentials.

- **Profile** — change your display name. Your email is the identity you signed
  up (or were invited) with and isn't editable here.
- **Change password** — requires your current password. A wrong current password
  is reported as exactly that, rather than signing you out. Changing it does not
  revoke your other sessions.
- **Active sessions** — every device signed in to your account (device, IP
  address, when it was created and last used). Revoke any of them to sign out
  elsewhere. See [Accounts & Workspaces](/docs/user-guide/workspaces).
- **Log out** — ends the current session server-side.

## Memory & privacy

The controls over what the platform remembers about you: pause or resume
learning, turn the use of existing memories on or off, export everything, and
forget everything. The list of individual memories lives on the **Memories**
page.

Memory is personal data rather than a workspace object, which is why its switches
live here. Full detail — including what is and isn't remembered, and who can read
each fact — is in [Long-Term Memory](/docs/user-guide/memory).

## Usage

Headline totals for the active workspace over a trailing **7, 30, or 90 days**:
requests, tokens, latency, and cost. It's the summary view — the per-agent and
per-model breakdowns and charts are on the
[Usage dashboard](/docs/user-guide/usage).

## Workspace settings

Workspace-level settings — renaming the workspace, members, and invitations —
are separate from your personal settings and live under **Workspaces**. See
[Accounts & Workspaces](/docs/user-guide/workspaces).
