---
sidebar_position: 1
title: Introduction
---

# Introduction

DAIRO AI Orchestrator is an **Agent-as-a-Service (AaaS) platform**: it lets you build,
configure, test, and serve AI agents as a managed service, rather than wiring up
prompts and integrations by hand for every project. The platform is made up of a
Python/FastAPI backend and a React/Vite frontend that together provide a single place
to create agents, wire them up to models and tools, and put them in front of real
users.

This guide introduces the core concepts you'll encounter throughout the rest of the
documentation.

## Core concepts

- **Agents** — The central object in the platform. Agents are created from templates
  and then configured with an agent pattern, guardrails, and other settings that
  control how they behave.
- **Agent Patterns** — Named, versioned recipes that define how an agent actually
  runs, such as `function_agent` and `react_agent`. A pattern maps to executable
  logic behind the scenes, so upgrading a pattern's implementation doesn't break
  agents that already use it.
- **Playground** — A streaming chat interface for testing an agent interactively
  before it's put into production, so you can see exactly how it responds to real
  messages.
- **Knowledge Bases** — Support Retrieval-Augmented Generation (RAG) by ingesting
  documents and making them searchable. Knowledge bases can be attached to one or
  more agents to ground their answers in your own content.
- **MCP Servers** — External tool servers (Model Context Protocol) that agents can
  call out to, extending what an agent can do beyond generating text.
- **Model Configs & BYOK** — Bring-your-own-key configuration for multiple LLM
  providers, including Anthropic, OpenAI, Gemini, and Bedrock, plus a built-in
  offline `fake` provider that requires no API key at all — useful for trying the
  platform out or running it without network access.
- **Model Routers** — Let you pick which underlying model handles a given request,
  so different requests can be routed to different models based on your own rules.
- **Observability** — Agent, tool, and LLM runs can be traced using Arize Phoenix,
  giving visibility into what an agent actually did during a conversation.
- **Workspaces** — Scope data to an organization. Today the platform runs in a
  single default workspace, but the data model is multi-tenant-ready, so
  organization-level isolation can be turned on without a redesign.

## How these docs are organized

These docs are split into three parts:

- **User Guide** (this section) — walks through using the platform: creating and
  configuring agents, testing them in the playground, working with knowledge bases,
  MCP servers, model configs, routers, guardrails, observability, and workspaces.
- **Developer Guide** — covers the platform's architecture and the conventions and
  workflow used to develop and extend it, for anyone contributing to the codebase.
- **API Reference** — the full HTTP API surface, generated from the platform's
  OpenAPI specification.

Use the navigation bar at the top of the site to switch between the three
sections.
