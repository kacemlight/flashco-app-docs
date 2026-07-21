# 4. Memory & Context Engineering

Context is the scarcest resource in a long-running agent system. A company
that operates for days will generate orders of magnitude more conversation
than any context window holds — and even before hitting limits, bloated
context degrades reasoning ("context rot") and multiplies cost. FlashCo's
answer is a layered memory architecture built on one principle:
**architectural prevention beats reactive compaction.**

## Three-tier conversation memory

Every agent holds private memory in three tiers:

### Tier 1 — Working memory
Recent turns at full fidelity, bounded by a token budget and a message
cap (with a minimum always retained). This is where active reasoning
happens.

### Tier 2 — Summary memory
When working memory overflows, the **oldest half** is compressed by a
cheap, fast model into a running summary (its own, smaller token budget).
Summarization cuts at safe points only — never splitting a tool call from
its result — and merges into the existing summary. If summarization itself
fails, an emergency trim hard-drops the oldest messages: degraded recall
beats a crashed agent.

### Tier 3 — Episodic memory
Structured facts auto-extracted from tool calls as they happen — "wrote
file X", "completed task Y", "delegated Z to Maya" — kept as a capped list
of one-liners. These survive any amount of summarization: the agent may
forget the conversation's texture but never what it *did*.

At prompt-build time the tiers compose: episodic facts + summary as a
context preamble, then the observation-masked working memory.

## Observation masking

The single highest-leverage technique in the stack. Tool outputs are the
#1 context flooder — a web page or file read can be thousands of tokens
that matter intensely for one turn and barely afterward.

Two-stage treatment:

1. **Compression at storage** — oversized tool results are truncated before
   they ever enter memory.
2. **Masking at recall** — outside the most recent turns, tool-result
   content is replaced with short stubs ("[output of web_browse — 4.2k
   chars, masked]"). The agent keeps the *narrative* of what it did; the
   bulky observations age out.

This mirrors findings from published agent research (the technique
originates in JetBrains' work on coding agents): reasoning continuity
survives masking almost perfectly, at a fraction of the tokens.

## Context isolation per agent

Agents **never share a raw conversation.** Each holds private memory;
information crosses between agents only through explicit channels —
delegation results (size-capped) and deliverables (files).

The rejected alternative was the classic group chat, where all agents see
all messages. Three reasons it lost:

1. **Token economics** — shared context grows linearly with conversation
   length *times* the number of agents reading it.
2. **Attributability** — with isolated memories, every decision has an
   owner and a traceable context; group chats blur causality.
3. **Published evidence** — shared visibility still produces conflicting
   decisions between agents; it doesn't buy the coherence it promises.

Isolation makes handoffs explicit and inspectable — which is also what
makes the evaluation judge's collaboration scoring
([doc 6](./06-evaluation-methodology.md)) meaningful.

## Blueprint-as-constitution + prompt caching

Every agent's system prompt is assembled from the rendered blueprint
documents (SOUL.md, AGENTS.md, TOOLS.md), the live integration status, the
discovered tool catalog, and the agent's persona and current task. Two
properties matter:

- **Behavior lives in reviewable documents**, not scattered hardcoded
  strings — the same artifact humans review is what the model reads.
- **The assembled prompt is cached** (provider-side prompt caching), so the
  large static prefix costs almost nothing per turn. Volatile facts go into
  conversation messages, never into the cached prefix.

## Queued-message consolidation

When an agent is busy, inbound messages queue (bounded per company). On
drain, queued messages are **consolidated into a single combined turn** —
five investor pings become one coherent instruction instead of five
context-fragmenting interruptions.

## Measured stance: prevention > compaction

The literature reports both compaction (summarize/prune as you go) and
isolation (never let context co-mingle) with isolation consistently
cheaper — the reference numbers show isolation cutting context cost
roughly 70% versus ~65% for aggressive compaction, with less reasoning
damage. FlashCo uses **both**, in that order of preference: isolation as
the architecture (per-agent memory, capped delegation results, files as
the inter-agent medium), compaction as the safety net inside each agent's
own lane.

## What's deliberately absent (for now)

**Long-term, cross-company memory.** Today an agent's memory dies with its
company — by design: it makes runs reproducible, keeps tenant data
isolated, and forces blueprints (not accumulated state) to carry the
intelligence. The planned evolution — vector-backed memory pools so a
blueprint's *N*th launch is smarter than its first — is on the
[roadmap](./09-roadmap.md), gated on the evaluation harness proving the
overhead pays for itself in outcome quality.
