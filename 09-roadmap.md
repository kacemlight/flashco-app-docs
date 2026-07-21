# 9. Roadmap & Known Gaps

An honest reference implementation publishes what it *hasn't* built with
the same care as what it has. This roadmap comes from mapping the
architecture pattern-by-pattern against the published multi-agent
literature; each item names the gap, why it matters, and what unlocks it.

## Reliability tier (next up)

### Per-step progress evaluation with intelligent retry
Today the task loop is dispatch → execute → done/review. The plan-based
orchestration literature closes the loop with a structured judgment after
each step — *completed? confidence? failure reason? suggested
improvements?* — and a retry that feeds the failure analysis back as
enhanced context. The schema-forced structured-output foundation this
needs is already in place; this is the highest-value reliability upgrade
remaining.

### LLM completion-check before loop exit
Agents can currently stop early with the loop none the wiser — the
"early stopping" failure mode. A judge pass over the agent's actual tool
calls and results before the loop may exit (with bounded restarts) closes
it.

### Composable middleware chain
The runtime's guard functions (injection defense, redaction, Rule of Two,
rate limits) work but are wired directly into the executor. Refactoring
them into an ordered, pluggable request/response/error pipeline makes the
security layer independently testable and extensible — each guard becomes
a module with a contract.

### Mid-turn cancellation
Pause today takes effect between executions. Cancellation tokens checked
before each model call and tool execution would make interruption
immediate — the interactivity standard for long-running work.

### Step-level checkpointing
Agent state currently persists on graceful shutdown and restores on boot;
a hard crash loses in-flight work. Persisting executor state after each
turn — with resume semantics that skip completed expensive calls — buys
crash-safe recovery and, eventually, horizontally scalable stateless
workers.

## Capability tier

### Blueprint-selectable orchestration modes
The hub-and-spoke dispatcher is today's single orchestration style. The
plan is to make orchestration a blueprint-level choice — rule-based /
AI-driven selection / plan-based — turning the wizard into a
pattern-selection surface and enabling empirical comparison of styles on
identical missions (the evaluation loop makes that comparison measurable).

### Long-term memory — "companies that learn"
Vector-backed memory pools scoped per creator or per blueprint, so the
*N*th launch of a design benefits from the first N−1. Deliberately gated
on the evaluation harness demonstrating that memory overhead pays for
itself in outcome quality — memory for its own sake is cost, not
capability.

### Token streaming to the dashboard
Agent messages currently appear whole. Streaming tokens to the live
dashboard is pure experience polish for the watching investor.

## Ecosystem tier

### OpenTelemetry traces
The event stream is rich but custom. Emitting Gen-AI
semantic-convention spans over the existing events would plug FlashCo
into standard observability tooling without re-instrumenting.

### A2A server facade
Exposing each company as an Agent-to-Agent server — an agent card at a
well-known URL, task lifecycle over the protocol — so external agents can
*hire a FlashCo company* as an opaque collaborator. The inverse of the
integration layer: today FlashCo consumes tools; this would make it one.

### Behavioral alignment evaluation
The judge scores outcomes; it does not yet probe behavior — sycophancy
under investor pressure, drift across model upgrades, conduct when
monitoring is (apparently) absent. The agentic-ethics literature is clear
that these need launch-blocking criteria of their own.

## What "done" looks like

It doesn't. The premise of a living reference implementation is that the
gap list never empties — it just gets more interesting. Each closed item
above will be documented in this series the way the shipped patterns are:
what was built, why that way, and what it changed in the measured
outcomes.

*Suggestions and critique: [kacem.aitoual@gmail.com](mailto:kacem.aitoual@gmail.com)
— or launch a company at [flashco.app](https://flashco.app) and see the
current state for yourself.*
