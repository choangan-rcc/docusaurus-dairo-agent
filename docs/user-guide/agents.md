---
sidebar_position: 3
title: Agents
---

# Agents

Agents are the central object in the platform: each one bundles a system
prompt, model settings, an agent pattern, tools, approval gates, and
guardrails. This page walks through creating and configuring agents in the web
UI; the same operations are available over the
[Agents API](/docs/api-reference/agents).

## Creating an agent

From the **Agents** page, create an agent either **from scratch** (only a name
is required — everything else has sensible defaults) or **from a template**.
Four templates ship with the platform:

- **Customer Support Agent** — front-line support with handoff-aware guardrails.
- **Internal Knowledge Agent** — answers questions from internal content.
- **Sales Assistant** — product questions and lead qualification.
- **Document Analyst** — works through document-heavy tasks.

A template pre-fills the system prompt, model settings, and guardrails; anything
you change on the form overrides the template's defaults. New agents always
start in the `draft` status.

## Configuring an agent

The agent form is organized into sections:

### Prompt and template variables

The **system prompt** defines the agent's persona and instructions. Prompts may
contain `{{variables}}` (e.g. `{{company_name}}`), whose values you fill in as
**template variables** — useful when several agents share one prompt shape.

### Model

Pick how the agent chooses its model — one of two mutually exclusive options:

- **A model config** — a specific provider + model + your own API key (see
  [Model Configs](/docs/user-guide/model-configs)). Agents without a pinned
  config use the server's environment-configured provider.
- **A model router** — routing rules that pick a model per request (see
  [Model Routers](/docs/user-guide/model-routers)).

Alongside that, set `temperature` (0–2), `max_tokens`, and the language.

### Pattern

The **agent pattern** decides how the agent actually runs:

- **Function Calling Agent** (`function_agent`, the default) uses the model's
  native tool-calling API — the best choice for models that support function
  calling.
- **ReAct Agent** (`react_agent`) drives a reason-then-act loop through text
  prompting — works with any chat model, at the cost of more prompt overhead.

Every pattern has two safety settings: **max iterations** per turn (default 10)
and a **timeout** per turn (default 120 seconds).

### Tools

Enable any of the built-in tools — `calculator`, `current_time`, and a set of
Yahoo Finance tools (balance sheet, income statement, cash flow, stock info,
analyst recommendations, news). New agents start with `calculator` and
`current_time`.

Each enabled tool also has a **"requires approval"** toggle. Tools marked this
way pause the conversation and wait for a human decision before every call —
see [Playground](/docs/user-guide/playground#human-in-the-loop-approvals).

Agents can get three more kinds of tools beyond this list:

- a **knowledge-base retrieval** tool — attach a KB (see
  [Knowledge Bases](/docs/user-guide/knowledge-bases));
- **data-source** tools — attach a connected database (see
  [Data Sources](/docs/user-guide/data-sources));
- **MCP** tools — attach an MCP server (see
  [MCP Servers](/docs/user-guide/mcp-servers)).

For those three, the approval gate lives on the **connection**, not here: a data
source or MCP server carries its own list of tools that require approval, so the
gate follows the connection to every agent that uses it.

### Memory

Two per-agent switches over [long-term memory](/docs/user-guide/memory):

- **Use memory** — off means this agent neither recalls nor learns facts, whatever
  the platform and personal settings say.
- **Isolated** — the agent keeps everything it learns in its own silo, and doesn't
  read the shared workspace-wide facts (language, addressing, timezone, response
  style). For sensitive domains.

### Guardrails

A per-agent on/off switch plus a fallback message used when a check blocks a
turn. The checks themselves (content safety, jailbreak detection, topic
control) are platform-level configuration — see
[Guardrails](/docs/user-guide/guardrails).

## Prompt version history

Every change to the system prompt or its template variables is snapshotted
automatically — no publish step needed. The agent's **Versions** tab lists the
history (version number, timestamp, changelog, and the full prompt snapshot),
and any older version can be **restored**. Restoring never rewrites history: it
copies the old prompt back and records a new version noting
"Restored from version N".

## Duplicating and deleting

- **Duplicate** deep-copies an agent's entire configuration into a new draft
  named "Copy of …" — handy for iterating on a production agent safely.
- **Delete** removes the agent. Its recorded usage is kept and folded into a
  "(deleted agents)" bucket on the [usage dashboard](/docs/user-guide/usage).

## Status

Agents carry a lifecycle status (`draft` when created; typically flipped to
`active` when ready). The Agents list can filter by status.
