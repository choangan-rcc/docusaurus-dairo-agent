---
sidebar_position: 6
title: Data Sources
---

# Data Sources

Data sources connect your agents to systems you already run — a PostgreSQL
database today — so an agent can look up tables, run queries, and summarize what
it finds. You pick a provider, enter your connection details, and the platform
does the rest.

**How this differs from an MCP server.** With an [MCP server](/docs/user-guide/mcp-servers)
you register a server you host and manage yourself. With a data source you supply
credentials only: the platform launches a curated, version-pinned connector for
that provider on your behalf. There is no command line to write and no server to
operate — and because the launch recipe is fixed by the platform, connecting a
database can never turn into running arbitrary code.

## Connecting a source

From the **Data Sources** page, choose **Connect** and walk the four steps:

1. **Pick a provider** — the catalog of what can be connected.
2. **Enter details** — the form is generated from the provider's own field list.
   For PostgreSQL that's host, port, database, username, and password, plus an
   access level.
3. **Test** — the platform starts the connector, checks your credentials, and
   asks the system what it can do. This takes a few seconds; a slow attempt
   explains itself rather than sitting silent.
4. **Outcome** — on success you get a plain-language summary of what your agents
   will be able to do with the connection.

### Access level

Every connection is **read-only** or **read and write**.

- **Read-only** (the default) — the connector cannot modify data at all. Nothing
  on this connection needs an approval step, because no call can change anything.
- **Read and write** — write-capable operations become available, and they are
  **automatically put behind human approval** when you switch a connection over.

Pick read-only unless an agent genuinely needs to change data. It is the
difference between a mistake being a wrong answer and a mistake being a modified
row.

## Capabilities, not tool names

A connected source is described by what an agent can *do* with it — "Look up
tables and columns", "Run queries and summarize results", "Change data (insert,
update, delete)" — rather than by a list of protocol tool names. The literal
tool list stays available behind **See details** when you want it.

## Statuses

| Status | Meaning |
| --- | --- |
| **Pending** | Saved, but not yet successfully checked. |
| **Connected** | The last check reached the system and listed its capabilities. |
| **Failed** | The last check failed. The reason is shown; fix a field and test again. |
| **Paused** | You turned the connection off. It isn't checked and agents don't use it. |

Only a connection that is **on** *and* **connected** is used in a conversation.
Anything else is quietly skipped — an agent never errors out because a source is
down; it simply doesn't have those capabilities for that turn.

A failed check is a state, not a lost form: your credentials stay stored, so you
adjust the host or the port and press **Test** without retyping the password.

:::note Why failure messages are short
The platform reports failures in a fixed set of phrases ("The connection was
refused or the host is unreachable"). Database drivers and connectors habitually
quote the full connection string — password included — in their error text, so
that text is never passed through. You get the classification, never the raw
message.
:::

## Your credentials

- Secrets are encrypted at rest and **never returned** by the API or shown in the
  UI. A connection displays only a masked hint of what's stored.
- Editing a connection doesn't require retyping secrets — leave a password field
  blank and the stored value is kept. Use **Replace credential** when you do want
  to change it.
- Credentials are decrypted only in memory, at the moment a connector is started
  for a turn.

## Attaching to agents

Attach a source to an agent from the agent's editor, or from the source itself.
Unlike an MCP server — where you must pick specific tools — a data source
defaults to **all of its capabilities**, since a database's tool list is derived
from its own schema rather than hand-registered. Use **Customize scope** to
narrow it.

Each connection's detail page lists the agents using it, which doubles as the
impact preview before you pause or delete it.

## Keeping a connection healthy

- **Test** re-checks the stored credentials and records the outcome.
- **Refresh capabilities** re-reads what the system offers — useful after a
  schema or permission change. It is the same operation as Test: listing
  capabilities and checking health happen in one session, so a refresh always
  also tells you the connection's real state.
- **Pause** stops the connection being used without deleting anything.
- **Delete** removes it and detaches it from every agent.

## API

The full endpoint reference — provider catalog, connection object, approval
gating, and error codes — is in
[Data Sources](/docs/api-reference/data-sources).
