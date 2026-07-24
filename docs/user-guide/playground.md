---
sidebar_position: 4
title: Playground
---

# Playground

The Playground is a streaming chat interface for testing an agent before (and
after) it goes in front of real users. It shows exactly what the agent does
during a turn: the streamed reply, the model's reasoning (when the model exposes
it), every tool call with its arguments and result, and — when routing or
tracing are enabled — which model served the turn and a link into the trace.

## Conversations and memory

Conversations are keyed by a session: keep chatting in the same conversation and
the agent remembers the previous turns; start a new conversation for a clean
slate. Only the most recent messages (40 by default) are replayed to the model
each turn — older context drops out of the prompt, but the full history stays
stored and browsable.

Conversations can be renamed and deleted from the conversation list; deleting a
conversation removes all of its messages permanently.

## What you see during a turn

- **Streaming text** — the reply renders token by token.
- **Reasoning** — models that expose reasoning stream it separately from the
  final answer.
- **Tool calls** — each call renders as a card with the tool name, the arguments
  the model chose, and the tool's output once it finishes.
- **Usage** — every turn records tokens in/out, latency, and estimated cost
  (visible on the [usage dashboard](/docs/user-guide/usage)).
- **Routing badge** — when the agent uses a
  [model router](/docs/user-guide/model-routers), each message shows which model
  served it and why (rule match, sticky session, or fallback).

## Human-in-the-loop approvals

If a tool is marked **requires approval** on the agent, the conversation pauses
just before that tool runs. An approval card appears showing the tool name and
the exact arguments the model wants to use, with **Approve** and **Deny**
actions (and an optional reason).

- **Approve** — the tool runs and the turn resumes streaming where it left off.
- **Deny** — the tool is *not* run; the model is told the call was denied (plus
  your reason, if given) and adapts its answer.

Pending approvals are durable — they survive a page reload or disconnect — but
expire after **15 minutes**; an expired approval asks you to resend the message.
The message composer is locked while a decision is outstanding.

## Guardrail blocks

When [guardrails](/docs/user-guide/guardrails) block a message (on input or
output), the agent replies with the configured fallback message instead, and the
turn is marked as guardrail-blocked. Both the original message and the fallback
reply are kept in the history for auditability.

## Using the API instead

Everything the Playground does is available over the HTTP API — including SSE
streaming and approval resolution — so you can build your own chat surface. See
[Chat / Playground](/docs/api-reference/chat) in the API reference.
