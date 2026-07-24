---
sidebar_position: 8
title: Model Routers
---

# Model Routers

A model router picks which model serves each request, instead of pinning an
agent to a single model. Typical uses: send long conversations to a bigger
model, route off-hours traffic to a cheaper one, or match keywords to a
specialist model — while keeping one fallback for everything else.

## How routing works

A router is an **ordered list of rules** plus a **required fallback** model
config. On every message:

1. **Rules run top-to-bottom** — the first rule whose conditions match picks the
   model for the turn. Reorder rules to set priority.
2. **Sticky sessions** — if no rule matches but the conversation recently used a
   routed model, it stays on that model for the sticky window (default 5
   minutes, refreshed on use). This prevents distracting model-hopping
   mid-conversation. A matching rule always beats stickiness.
3. **Fallback** — otherwise the fallback model serves the turn.

A router with no rules behaves like a plain model pin (everything goes to the
fallback).

## Building rules

Each rule targets one model config and combines one or more conditions with
AND/OR:

| Condition | Example |
| --- | --- |
| **Prompt content** | contains any of `refund, cancel, chargeback`; or matches a regex |
| **Conversation token count** | greater than `4000` → send to the long-context model |
| **Conversation message count** | between `1,3` → cheap model for short exchanges |
| **Current hour** (UTC) | between `22,6` → overnight traffic to the budget model (wraps past midnight) |

Rules are validated when you save — a bad regex, an out-of-range hour, or a
reference to a deleted model config is caught immediately.

## Safety checks

- **Require function calling** (on by default): if the router will serve agents
  that use tools, saving rejects any target model known not to support function
  calling. The same check runs when you assign a router to a tool-enabled agent.
- **Needs-repair badge**: if a model config referenced by a rule or the fallback
  is deleted later, the router card shows a repair badge. Rules pointing at a
  deleted config are skipped at runtime; a deleted *fallback* must be fixed
  before the router can serve turns.

## Assigning to an agent

In the agent form's model picker, routers appear alongside model configs
(labeled "router · N rules"). Picking a router clears any pinned model config
and vice versa — an agent has exactly one model source.

## Seeing routing decisions

In the [Playground](/docs/user-guide/playground), each routed message carries a
badge showing which model served it and why — a named rule, a sticky session, or
the fallback. The same decision lands in the message trace and (when tracing is
on) in the [observability](/docs/user-guide/observability) trace metadata.

## API

Full rule schema, evaluation semantics, and endpoints are in
[Model Routers](/docs/api-reference/model-routers).
