# 1. System Architecture

FlashCo is a distributed system with one governing principle: **the control
plane and the agent runtime are different machines with different jobs.**
Agents are powerful and unpredictable; everything that meters, gates, or
audits them lives outside their blast radius.

## The topology

```
┌─────────────────┐   REST    ┌──────────────────┐  webhooks (HMAC)  ┌──────────────────┐
│   Web (Next.js) │──────────▶│  Engine (Python)  │◀──────────────────│ Gateway (Node.js) │
│  UI · wizard ·  │           │  control plane:   │──────────────────▶│  agent runtime:   │
│  OAuth · billing│           │  lifecycle · cost │   REST (deploy)   │  LLM loops · MCP  │
└────────┬────────┘           │  metering · evals │                   │  memory · tools   │
         │                    └────────┬─────────┘                    └────────┬─────────┘
         │        realtime             │      all LLM calls (metered proxy)    │
         ▼                             ▼                                       ▼
┌──────────────────────────────────────────────┐            ┌────────────────────────────┐
│    Postgres — auth · realtime · storage      │            │  Stateless MCP servers     │
│    (single source of truth)                  │            │  (JSON-RPC over HTTP)      │
└──────────────────────────────────────────────┘            └────────────────────────────┘
```

Four deployable services plus a managed database:

### Web — the experience layer

Next.js application serving three audiences: **investors** (people who launch
and fund companies), **creators** (people who design blueprints), and
**visitors** (the educative landing experience). It owns the blueprint
wizard, the launch flow, OAuth authorization-code exchanges for
integrations, and the live dashboard. It renders blueprints into concrete
workspaces at launch time (placeholder substitution, credential encryption)
but **never talks to an agent directly** — everything operational goes
through the engine's API.

### Engine — the control plane

A Python service that owns everything about a company *except* running its
agents:

- **The lifecycle state machine** — `draft → funding → deploying → active →
  delivering → dissolved`, with each transition owned by explicit code, not
  agent judgment.
- **The credit ledger** — deposits, deductions, refunds, and the
  budget-strike thresholds (see [doc 8](./08-economics-and-lifecycle.md)).
- **The LLM metering proxy** — every model call from every agent flows
  through one endpoint that translates between providers, retries rate
  limits, and records tokens/cost/latency per request. Agents cannot call a
  model without being metered.
- **The evaluation judge** — scores dissolved companies
  ([doc 6](./06-evaluation-methodology.md)).

### Gateway — the agent runtime

A Node.js service where agents actually live:

- Per-company **isolated workspaces** (own directory, own memory, own MCP
  connections, own scheduler).
- The **agent execution loop** — system-prompt assembly, the tool-call loop
  with turn caps and task timeouts, and the three-tier conversation memory
  ([doc 4](./04-memory-and-context.md)).
- The **tool sandbox** — native tools (web search/browse, deliverables,
  delegation, approvals) plus MCP clients, all passing through the security
  guard ([doc 5](./05-security-model.md)).
- **Schedulers and watchers** — per-company task dispatch loops, scheduled
  heartbeat briefings, and a deadline watcher that dissolves overdue
  companies.

### Stateless MCP servers — the tool surfaces

Self-hosted tool servers (e.g. a headless-CMS publisher, a document-suite
server) speaking JSON-RPC 2.0 over HTTP (the Model Context Protocol's
streamable transport). Their defining property: **they hold zero
credentials**. The gateway forwards a tenant's token with every request.
One deployment serves every tenant, scale-to-zero keeps them nearly free,
and the confused-deputy vulnerability class disappears by construction
([doc 7](./07-integration-layer-mcp.md)).

## The event-sourced state mirror

The single most load-bearing architectural decision: **the gateway persists
nothing user-facing directly.**

Every significant runtime event — agent message, status change, tool use,
task start/completion, deliverable written, session start/end — is emitted
as an **HMAC-signed webhook** to the engine. The engine validates,
prices (tool calls carry micro-costs), and mirrors the event into
Postgres. The database's realtime channel then fans changes out to the
dashboard.

```
gateway (behavior) ──HMAC webhook──▶ engine (validate + meter) ──▶ Postgres ──realtime──▶ dashboard
```

What this buys:

- **One-directional data flow.** State has a single writer per concern; the
  dashboard is always a projection, never a participant.
- **A complete audit log for free.** The webhook stream *is* the activity
  history — which later becomes the evaluation trajectory.
- **Metering as a side effect of observation.** Costs are recorded where
  events are validated, so an agent cannot act without being priced.
- **Restartability.** The gateway can be redeployed; the durable record
  lives elsewhere. Agent conversation state is serialized to disk on
  graceful shutdown and restored on boot.

## Trust boundaries

| Boundary | Mechanism |
|---|---|
| Web → Engine | Bearer secret |
| Engine ↔ Gateway | Bearer token (REST) + HMAC-SHA256 signatures (webhooks) |
| Gateway → MCP servers | Per-request forwarded OAuth/bearer tokens + tenant context headers |
| Browser → Database | Row-level security under the user's auth token; services use a privileged key server-side only |
| Agents → LLM providers | Only via the metering proxy; BYOK keys are decrypted per request, never entering agent context |

## Why three tiers instead of one process

A single-process agent framework is simpler until the first incident. The
split earns its complexity three ways:

1. **Blast-radius control.** An agent runtime bug (runaway loop, memory
   leak, hostile tool output) cannot corrupt the ledger or lifecycle state —
   those live in a different service with a different write path.
2. **Independent scaling and deployment.** The runtime redeploys many times
   a day during pattern work; the control plane changes rarely. Coupling
   them means every experiment risks the money path.
3. **Honest metering.** Cost and audit records are only trustworthy when
   the thing being measured cannot write them. Agents emit events; the
   control plane prices them.

## Related docs

- Blueprint rendering pipeline → [doc 2](./02-blueprint-notation.md)
- How work is dispatched inside a company → [doc 3](./03-orchestration-patterns.md)
- The lifecycle state machine in full → [doc 8](./08-economics-and-lifecycle.md)
