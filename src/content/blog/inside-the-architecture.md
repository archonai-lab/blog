---
title: 'Inside the Architecture: How Archon Actually Works'
description: 'WebSocket hub, per-agent brains, structured meetings with relevance-based turns. A technical walkthrough of the Archon stack.'
pubDate: 'Mar 29 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

People ask "what is Archon?" and we say "it organizes AI agents like a company." That's the pitch. Here's what it actually looks like under the hood.

## The stack

```
┌─────────────────────────────────────────────────────┐
│                  Archon Platform                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐ │
│  │  Hub (TS)    │  │ Postgres │  │ Neural Memory │ │
│  │  WebSocket   │  │ Company  │  │ (per-agent)   │ │
│  │  Auth/Route  │  │ DB       │  │ Graph-based   │ │
│  └──────┬───────┘  └──────────┘  └──────────────┘ │
│         │                                           │
│  ┌──────┴──────────────────────────────────────┐   │
│  │           Agent Runtime Layer                │   │
│  │  ┌─────┐  ┌──────┐  ┌────────┐  ┌───────┐  │   │
│  │  │ CEO │  │ Rune │  │Sherlock│  │ Kalyx │  │   │
│  │  └─────┘  └──────┘  └────────┘  └───────┘  │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

Three layers, cleanly separated.

## The hub: WebSocket, not MCP

This was our first big architecture decision. MCP (Model Context Protocol) is great for tool integration — it's how agents talk to memory, identity, and external services. But MCP uses stdio or SSE transport. It's point-to-point. It can't broadcast.

Meetings need broadcast. When Agent A speaks, every other participant needs to hear it in real time. When a phase transitions from "discuss" to "decide," every agent needs to know simultaneously.

So the hub is a purpose-built WebSocket coordination server. Every message follows a discriminated union pattern — `{ type: "meeting.speak", ... }` — validated by Zod at the boundary. The hub doesn't know or care what LLM an agent uses. It routes messages, manages sessions, and enforces meeting rules.

```typescript
// Every inbound message is validated against one schema
const InboundMessage = z.discriminatedUnion("type", [
  AuthMessage,
  PingMessage,
  MeetingSpeakMessage,
  MeetingRelevanceMessage,
  // ...
]);
```

Agents authenticate with a token, get a session, and then they're in. The hub tracks who's online, who's in which meeting, and handles graceful cleanup when agents disconnect.

## Agents connect, they're not spawned

Most multi-agent systems spawn agents as subprocesses. The orchestrator starts them, feeds them work, collects output. Archon does the opposite.

Agents bring their own brains and connect to the hub via WebSocket. The hub doesn't spawn or manage agent processes. An agent could be running on your laptop, a remote server, or a cloud function. The hub doesn't care — it just sees a WebSocket connection that authenticated as `sherlock`.

This decoupling is deliberate. It means agents can use any LLM provider (Claude, Gemini, GPT, local models) without hub changes. It means you can run an agent in Claude Code during development and move it to an API call in production. The hub focuses on coordination, not execution.

## Per-agent brains via archon-agent

Each agent gets its own MCP server called archon-agent. It provides three categories of tools:

**Identity** (1 tool) — `identity_load` reads the agent's SOUL.md and IDENTITY.md. The agent adopts the personality, tone, and constraints described. This runs once at session start.

**Memory** (8 tools) — Bridged from Neural Memory. `nmem_remember`, `nmem_recall`, `nmem_recap`, `nmem_context`, `nmem_session`, `nmem_explain`, `nmem_edit`, `nmem_forget`. The bridge is a dumb passthrough — archon-agent spawns a Neural Memory MCP process as a child and forwards tool calls.

**Meeting** (3 tools) — `meeting_send`, `meeting_receive`, `meeting_observe`. These connect the agent to the hub's meeting system.

The key design constraint: **agent brains are NOT shared**. Like humans don't share actual brains. Agent A's memories are invisible to Agent B. They share information by speaking in meetings, not by reading each other's memory.

## Meetings: structured, not free-form

A meeting in Archon has phases. The default methodology runs: **present** (agenda laid out), **discuss** (open floor), **decide** (proposals and votes), **assign** (action items distributed).

But here's the thing — methodologies are user-defined markdown files. You can create your own:

```markdown
# Code Review

## Phase: present
- capability: initiator_only
- budget: 6000
Present the diff and automated check results.

## Phase: analyze
- capability: open_discussion
- budget: 15000
Review the code. Flag issues.

## Phase: verdict
- capability: proposals
- budget: 8000
Approve, request changes, or block.
```

The hub reads these at runtime. No hardcoded phase enums.

### Relevance-based turns

This is the part that makes meetings actually work. In most multi-agent setups, every agent speaks every round. Five agents, five responses, most of them redundant.

Archon asks each agent: "Based on the discussion so far, how relevant is this to you?" The agent responds with one of three signals:

- **MUST_SPEAK** — I have something critical to add
- **COULD_ADD** — I have a minor point
- **PASS** — Nothing to add

MUST_SPEAK agents go first. COULD_ADD agents go next if budget allows. PASS agents stay silent. When everyone passes, the phase auto-advances.

The result: meetings that converge instead of sprawl. The security agent speaks when there's a security issue. The performance agent speaks when there's a performance issue. Nobody wastes tokens restating what someone else already said.

## The database

Nine Postgres tables model the company:

- **agents** — identity, status, model config, workspace path
- **departments** and **roles** — organizational structure
- **meetings** — phase, budget, agenda, decisions, action items
- **meeting_participants** and **meeting_messages** — who said what, when
- **permissions** — resource-level access control with wildcards
- **projects** — work tracking with methodology selection

Everything is typed end-to-end. Drizzle ORM schema in TypeScript, Zod validation at WebSocket boundaries, inferred types everywhere. No hand-written interfaces.

## What's next

~~The hub and brain are production-ready for identity + memory. Meeting integration in archon-agent is next.~~

**Update (2026-03-29):** Meeting integration is shipped. `meeting_send`, `meeting_receive`, and `meeting_observe` are live — backed by a lazy WebSocket connection with a capped message buffer and fail-fast reconnect. A minimal agent runner (`scripts/runner.ts`) drives agents through meetings via LLM, with per-meeting analytics tracking tool call latency, LLM response times, and relevance decisions.

The current focus: observability instrumentation (spanId correlation, prompt context logging) and a heartbeat protocol so the hub can detect dead agents. After that: 1-on-1 chat — DM your agents like teammates on Slack.

[GitHub](https://github.com/archonai-lab) | [Docs](https://archonai-lab.github.io/archon/)
