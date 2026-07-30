---
sidebar_position: 11
title: Long-Term Memory
---

# Long-Term Memory

By default an agent only remembers the conversation it's in. Long-term memory
lets it carry durable facts about you *between* conversations: say once that you
prefer replies in Vietnamese, or that "the analytics DB" means a particular
database, and the next conversation already knows.

Memory is **per person, per workspace**. Facts learned about you are yours: they
are never visible to other members, and never cross into another workspace.

## What gets remembered

After a conversation goes quiet, the platform reviews the new messages and pulls
out facts about you that are worth keeping — a stated preference, a decision
made, a goal set, a correction, or a new thing in your world (a project, a team,
a customer, a tool). Small talk yields nothing, which is the common case.

Each memory is one of two kinds:

| Kind | Meaning |
| --- | --- |
| **Ongoing fact** | Doesn't expire. Replaced if you correct it. |
| **Point in time** | Tied to a moment; becomes less relevant as it ages. |

Instead of piling up duplicates, a new fact that contradicts an old one
**replaces** it, and a paraphrase of something already known is dropped.

### Who can read a memory

This is the part worth understanding, because it's the isolation guarantee:

- **Every agent in this workspace** — a small set of identity facts: your
  language, how to address you, your timezone, and your preferred response style.
  You state these once and every agent knows them.
- **One specific agent** — everything else. A preference you mention to your
  support agent stays with that agent.

The reader list is decided by the platform, never by the model, so a
misclassified fact lands in the narrower silo — never in the shared one. An agent
can also be configured as **isolated**, which keeps everything it learns entirely
to itself.

### What is never stored

Passwords, API keys and tokens, private keys, and card numbers are excluded by
rule, along with payment data, government ids, health data, and precise home
addresses. Anything you ask to keep off the record is skipped.

Conversation text is treated as **data, not instructions** — a message that tries
to plant a rule in memory can at most be recorded as "the user said this", never
followed.

## The Memory Center

The **Memories** page lists everything the platform remembers about you in the
current workspace, and it's also where you fix things.

- **Filter** by agent, kind, reader list, or a text search.
- **Open a memory** to see its provenance: which conversation it came from, when
  it was learned, and who can read it.
- **"This is wrong"** offers exactly two honest repairs: **delete** the memory, or
  **tell the agent** the correct thing in chat so it learns the update. There is
  deliberately no edit box — editing would have to be delete-and-recreate, which
  would silently rewrite the memory's origin and date.
- **Select several** memories to delete them in one go.
- **Export** downloads everything remembered about you as a file.

Arriving from a [trace](/docs/user-guide/observability) with a list of memory ids
shows exactly the memories that turn used, in the order they were injected — with
any that have since been deleted reported rather than quietly missing.

## Turning memory on and off

There are four levels of control, and **all of them** have to allow it:

| Level | Where | What it controls |
| --- | --- | --- |
| Platform | Environment configuration | Whether memory exists in this deployment at all, and for which workspaces. |
| You | **Settings → Memory & privacy** | Whether new facts are learned, and whether what's already known is used. |
| Agent | Agent editor | Whether this agent uses memory, and whether it's isolated from the shared facts. |
| Conversation | The chat itself | Whether this one conversation reads memory, and whether it's learned from. |

Your two personal switches are independent:

- **Pause learning** — nothing new is saved. Nothing is deleted, and what's
  already remembered keeps being used. Resuming does **not** back-fill the gap:
  anything said while paused is never learned.
- **Use what's remembered** — turn this off to answer without memory anywhere in
  the workspace, while still keeping (and continuing to learn) the facts.

Per-conversation, the same two switches apply to just that chat. Turning the
write switch back on also skips whatever was said while it was off.

If your deployment can't learn — memory disabled, workspace not enabled, or no
background worker configured — the settings page says so plainly rather than
showing a switch that would promise saving that never happens.

## Forget everything

**Forget everything** hard-deletes every memory about you in this workspace and
turns learning off until you turn it back on. It's the right-to-forget path: the
rows and their embeddings are gone, not hidden. The count of what was deleted is
reported back.

## In a conversation

When memory is on, relevant facts are looked up as your message arrives and added
to the agent's context as reference material — explicitly framed as context, not
instructions, so that if you contradict a remembered fact, what you say now wins.

Two properties keep it out of the way:

- **It never slows a turn down noticeably.** Retrieval is bounded by a short
  timeout, and if anything is slow or fails, the turn simply proceeds without
  memory.
- **It never blocks a turn.** Learning happens in the background after the reply,
  so it can't fail your message.

Only facts genuinely related to what you asked are used — an unrelated question
injects nothing at all, rather than the least-irrelevant handful.

## API

The full endpoint reference — listing, filtering, export, bulk delete,
right-to-forget, the state object, and every configuration flag — is in
[Memories](/docs/api-reference/memories).
