# 17 — Orchestration Modes: Four Topologies Behind One Seam

> How FlashCo runs hub-and-spoke, peer-to-peer relay, task auctions, and
> consensus panels as swappable strategies — and measures which topology
> fits which kind of mission.

Every multi-agent framework picks a coordination topology and hard-codes it.
FlashCo treats topology as a **variable under test**: a blueprint selects its
orchestration mode, the runtime executes it, and every dissolved company's
evaluation is stamped with the mode it ran. The research question — *which
topology fits which mission type?* — accumulates an answer with every launch.

## The registry: one source of truth

A mode is **data first**. The shared registry defines everything the product
needs to know about a mode — copy, team contract, runtime availability:

```ts
// packages/shared/src/orchestration.ts
export interface OrchestrationModeDefinition {
  key: string;
  name: string;
  shortDescription: string;
  longDescription: string;
  advantages: string[];
  considerations: string[];   // honest trade-offs, shown in the wizard
  bestFor: string[];
  runtime: "live" | "planned";
  teamContract: {
    requiredRoles: string[];  // hub requires "pm"; peer modes require none
    minAgents: number;        // consensus needs 3
    compositionHints: string[];
  };
}

export function resolveOrchestrationMode(value: unknown): string {
  if (typeof value === "string" && value in ORCHESTRATION_MODES) return value;
  return DEFAULT_ORCHESTRATION_MODE; // unknown/missing values can never crash
}
```

Three rules keep the system extensible:

1. **No DB enum.** `orchestration_mode` is plain text on `templates`,
   `companies`, and `company_evaluations` — a new mode never needs a
   migration.
2. **One sanctioned reader.** Every consumer resolves through
   `resolveOrchestrationMode()`, so old rows and new modes are mutually
   harmless.
3. **Availability is data.** A mode can be selectable in the wizard while its
   runtime is `"planned"` — clearly labeled, running as the default until its
   strategy ships.

## The strategy seam

The scheduler owns everything mode-agnostic: polling, the dependency gate,
the concurrency cap, in-flight dedup, jitter, heartbeats. A strategy owns
exactly five decisions:

```ts
// apps/gateway/src/runtime/orchestration/types.ts
export interface OrchestrationStrategy {
  key: string;
  /** Which dependency-gated tasks fire this tick, in what order, to whom.
   *  May be async — auction modes run bid rounds before dispatching. */
  buildDispatchWave(ctx: WaveContext): TaskContext[] | Promise<TaskContext[]>;
  /** What happens when the board is quiet. */
  onIdle(ctx: IdleContext): Promise<void>;
  /** Optional narrative beat when a wave opens. */
  onWaveDispatched?(ctx: WaveDispatchedContext): Promise<void>;
  /** Who receives untargeted investor messages. */
  pickMessageReceiver(agents: AgentContext[]): AgentContext | undefined;
  /** Who may create tasks (hub: PM/CEO; peer modes: everyone). */
  canCreateTasks(role: string): boolean;
}
```

This boundary is what makes the measurements honest: memory, security
guards, tool filtering, budgets, and deadlines are **identical across
modes**, so score differences are attributable to the topology, not to
implementation accidents.

Adding a fifth mode = one registry entry + one strategy file.

## The four modes

### 1. Hub & Spoke (PM-as-dispatcher)

The PM sequences the task graph and dispatches dependency waves; specialists
report back. The scheduler backstops the hub — unblocked tasks dispatch even
when the PM is busy, and a watchdog re-kicks a PM that never spoke.
*Trade-off: one accountable narrative thread, but the hub is a latency
bottleneck and a bad plan propagates.*

### 2. Peer-to-Peer Relay

Kickoff broadcasts to every agent. Peers claim tasks in their own lanes and
hand cross-lane work directly to the owning teammate. Idle time runs a
**rotating standup** — each cooldown window, the next idle agent (round-robin)
reconciles the board and updates the investor, so narrative leadership
rotates through the team. *Trade-off: near-zero dispatch latency, but
ownership lanes must be airtight or work is dropped or duplicated.*

### 3. Task Auction

When a task unblocks, idle agents submit **structured bids** — a
`tool_choice`-forced schema on the cheap model, so a malformed bid is
impossible:

```ts
// The award is deterministic: honest confidence, weighted by role fit,
// effort as a tie-breaker. Role fit discounts bravado outside the lane.
function scoreBid(bid: TaskBid, agent: AgentContext, task: TaskContext): number {
  const roleFit =
    role === assigned ? 1.0 :
    role.includes(assigned) || assigned.includes(role) ? 0.85 :
    0.6;
  const effortPenalty = { low: 0, medium: 0.02, high: 0.05 }[bid.estimated_effort];
  return bid.confidence * roleFit - effortPenalty;
}
```

The winner executes with **its own winning plan as the brief**, and every
auction posts its audit trail to the investor chat: who bid what, who won,
why. *Trade-off: a small token/latency overhead per task, and auctions
degenerate when capabilities don't overlap — one plausible bidder is not a
market.* Bounds: two auctions per tick, four bidders each; dry markets fall
back to owner dispatch.

### 4. Consensus Panel

Each task runs a three-stage panel:

```
PROPOSALS  → up to 3 idle panelists work the task INDEPENDENTLY, each
             writing a complete proposal file, blind to the others.
             The task stays 'pending' — the board never lies.
SYNTHESIS  → when all proposals exist (or a 20-min timeout fires with at
             least one), the strongest available agent reads every proposal,
             merges the best ideas into the final deliverable, and credits
             what came from where.
FALLBACK   → teams too small for a panel, or a panel that produces nothing,
             degrade to plain dispatch. Quality mode never stalls a mission.
```

*Trade-off: K× tokens and added latency per task — the price of
uncorrelated attempts. The synthesis quality caps the panel's value.*

## Closing the loop: measurement

The one line that turns all of this into a research instrument:

```python
# apps/engine/src/evaluation.py — every dissolved company's judge scorecard
row = {
    "company_id": company_id,
    "overall": overall,                       # weighted 5-dimension score
    "scores": {...},
    "orchestration_mode": resolve_orchestration_mode(
        trajectory["company"].get("orchestration_mode")
    ),
}
```

Same blueprint, same mission, four topologies — scored by the same
five-dimension judge (deliverable quality 35%, task completion 25%,
collaboration 15%, efficiency 15%, communication 10%). The creator analytics
dashboard aggregates retrospectives by mode; as runs accumulate, the
mode × mission-type efficiency map stops being opinion.

## Design lessons

- **Topology is a strategy, not an architecture.** Once the seam existed,
  each new mode was one file. The hard work was extracting hub-and-spoke
  without changing its behavior — refactor first, verify, then diverge.
- **Structured output composes with coordination.** The same
  schema-forcing that makes task plans reliable makes *bidding* reliable.
- **Every mode needs a degradation path.** Panels without panelists,
  auctions without bidders, hubs with silent PMs — each falls back to plain
  dispatch instead of stalling. A quality or efficiency mode that can wedge
  a mission is worse than no mode at all.
- **Keep the board honest per mode.** Consensus keeps tasks `pending` during
  blind drafting because nothing is "in progress" until synthesis owns it;
  auctions mark tasks only when awarded. The investor's view never lies.
