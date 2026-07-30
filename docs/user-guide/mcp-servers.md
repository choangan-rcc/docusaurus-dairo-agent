---
sidebar_position: 7
title: MCP Servers
---

# MCP Servers

MCP servers extend what your agents can do beyond the built-in tools. Any
server that speaks the [Model Context Protocol](https://modelcontextprotocol.io)
can be registered once and then attached to any number of agents — each agent
choosing exactly which of the server's tools it may use.

## Registering a server

From the **MCP Servers** page, register a server with a name and one of three
transport types:

- **Streamable HTTP** or **SSE** — remote servers reached over a URL, with
  optional extra headers (for auth) and a connection timeout.
- **stdio** — a local process the platform launches itself (a command such as
  `npx`, plus arguments and environment variables).

Registration only stores the record — nothing connects yet. The server starts in
the `pending` state.

## Discovery

Run **Discover** on the server to connect, perform the MCP tool-listing
handshake, and cache the server's tool catalog. On success the server becomes
`active` and its tools (name, description, input schema) are listed on its page.
If the connection fails, the server is marked `error` — fix the config and
discover again.

Two things worth knowing:

- Discovery is the **only** operation that talks to the server ahead of time.
  Day-to-day, agents read the cached catalog; a live connection opens only at
  the moment a tool is actually invoked.
- **Editing a server's config resets it to `pending`** and clears its cached
  tools — run discovery again before agents can use it.

## Attaching to agents

Attach a server to an agent by picking an explicit, non-empty subset of its
discovered tools — there is no "attach everything" option, which keeps each
agent's tool surface deliberate. Re-attaching the same server updates the tool
selection in place.

At runtime, the agent's effective tools are its built-in tools plus the selected
tools from each attached **active** server. If a server is unreachable when the
model calls one of its tools, only that tool call fails (the model is told and
adapts) — the conversation keeps going, and the server is flagged `error` for
attention.

## API

The full endpoint reference — including config shapes per transport and error
codes — is in [MCP Servers](/docs/api-reference/mcp-servers).
