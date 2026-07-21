# 3. Orchestration Patterns

Orchestration answers one question: **who decides what happens next?**
FlashCo's answer is a deliberate hybrid — a deterministic control spine
with autonomous agents inside their lanes. This doc describes each pattern,
why it was chosen, and where full autonomy was deliberately *not* used.

## Hierarchical org chart

Agents have authority tiers, not just names:

- **PM** — owns the task plan and dispatch; primary contact for the
  investor; tiebreaker in conflicts.
- **CEO** — escalation authority; compiles the final delivery and writes
  the closing quality assessment at dissolution.
- **Specialists** — execute within their ownership lane.

Authority is enforced, not decorative: only the PM and CEO can create
tasks. Model tiers follow the hierarchy (the dispatcher and workers get the
capable model; cheap utility work gets the fast one).

**Why:** flat agent swarms stampede — everyone acts at once and work
collides. A hierarchy gives the system a deterministic spine while agents
remain autonomous inside their roles. It also matches human intuition:
investors instantly understand "ask the PM."

## PM-as-dispatcher (hub and spoke)

On kickoff, the PM reads the mission (SOUL.md), decomposes it into a task
graph, and dispatches work **in dependency waves**. Non-PM agents start in
standby: they act when assigned, and report back to the hub when done —
they do not self-start. Investor messages route through the PM first.

The runtime scheduler is built around this contract: it fetches pending
tasks, matches them to idle agents by role, and enforces wave ordering plus
a concurrency cap. When a blueprint has no PM, the scheduler's priority
waves still impose order — but the PM pattern is the recommended and
default design.

**Why:** dependency-ordered dispatch prevents the most common failure of
naive multi-agent systems — the report being written before the research
exists.

## The task graph

Tasks carry `priority` and `depends_on` edges (backward references only,
validated at generation via JSON schema — cycles are impossible by
construction). The scheduler dispatches a task only when its predecessors
have completed. Priorities order independent work; dependencies express
genuine handoffs. The distinction is explicit in the generation prompt:
*"do NOT add depends_on just to serialize independent work."*

## Handoff — agents as tools

`delegate_to_agent` is the peer-to-peer channel: it runs a **full nested
execution loop** on the target agent and returns their answer (capped in
size — a deliberate result-informativeness control that keeps delegation
from flooding the caller's context).

Target resolution degrades gracefully: exact role → exact name → substring
→ word-overlap fuzzy match. Agents address colleagues the way humans do —
imprecisely — and the runtime absorbs it.

**Why:** work crosses ownership lines constantly. Delegation moves context
between specialists without forcing every exchange through a central
bottleneck — and without a shared group chat (see
[doc 4](./04-memory-and-context.md) for why shared conversations were
rejected).

## Message routing and queueing

Inbound investor messages route through a priority chain: explicit
`@mention` target → PM → CEO → any idle agent → a bounded per-company
queue. Queued messages are **consolidated** when the agent frees up — a
burst of investor pings becomes one coherent turn instead of five
fragmented ones.

## Termination — five concurrent conditions

Knowing when to stop is where autonomous systems most often fail in
production. FlashCo runs five termination layers simultaneously:

| Condition | Mechanism |
|---|---|
| **Semantic** | The agent declares completion via a `complete_task` tool call (with a result summary) |
| **Turn cap** | Hard per-execution limit on LLM loop iterations |
| **Task timeout** | Soft warning injected at ~6 minutes ("wrap up now, ship partial work"), hard stop at 8 |
| **Budget** | The strike system: escalating thresholds up to a full stop ([doc 8](./08-economics-and-lifecycle.md)) |
| **Deadline** | A watcher broadcasts "compile your deliverables" as the company's end time passes, then dissolves it |

The design principle: **terminating early with partial results beats
running forever.** Every layer prefers shipping something over burning
budget on perfection.

## Temporal patterns

- **Heartbeats** — scheduled morning/evening briefings requested from the
  PM (configurable via HEARTBEAT.md). The company has a circadian reporting
  rhythm; the investor hears from it without asking.
- **Idle-time reflection** — when no tasks are pending and the PM has been
  idle past a cooldown, the scheduler asks it to review progress and post a
  status update. A lightweight metacognition loop.

## What was deliberately *not* automated

Two orchestration styles from the literature are intentionally absent (for
now — see [roadmap](./09-roadmap.md)):

- **LLM-as-orchestrator** (a model choosing the next actor each turn):
  rejected as the default because coordination-by-inference adds a model
  call per hop, is hard to audit, and most missions have decomposable
  structure that rules dispatch better. Planned as a blueprint-selectable
  mode, not a replacement.
- **Shared-conversation group chat**: rejected for token economics and
  attributability — covered in [doc 4](./04-memory-and-context.md).

The stance throughout: **autonomy is a dial.** Deterministic where
structure is known (dispatch, routing, termination), autonomous where
judgment is needed (how to do the work, when to delegate, when to ask a
human).
