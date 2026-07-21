# 8. Economics & Lifecycle

The least-discussed patterns in multi-agent literature are the ones that
decide whether a system survives contact with production: **budgets,
termination, and mortality.** FlashCo treats economics as a first-class
orchestration mechanism — not billing plumbing bolted on afterward.

## The company state machine

```
draft → funding → deploying → active ⇄ (paused) → delivering → dissolved
```

Every transition is owned by explicit control-plane code — never by agent
judgment:

- **draft → deploying**: the blueprint renders against the mission,
  credentials are encrypted, agents are registered, the workspace is
  created, initial tasks seed the graph, and the PM receives its kickoff.
- **active**: the operating state — schedulers dispatch, agents work,
  events mirror to the dashboard.
- **paused**: reversible suspension — investor-initiated or automatic
  (budget strike).
- **delivering → dissolved**: deliverables are compiled (the CEO owns the
  final pack and quality assessment), unused budget is refunded, agents
  are terminated, the workspace is torn down, and the retrospective judge
  runs ([doc 6](./06-evaluation-methodology.md)).

## Deliberate mortality

Companies are **born to dissolve**. A deadline is set at launch; a watcher
converts its passing into an urgent "compile your deliverables" broadcast
and then a forced dissolution. Three reasons mortality is a feature:

1. **Convergence pressure.** Teams that never end optimize for staying
   busy; teams with an end date optimize for shipping.
2. **Bounded liability.** No zombie agents, no forgotten workers burning
   tokens at 3 a.m., no unbounded accumulation of state.
3. **Clean experimental units.** A bounded run with a beginning, an end,
   and a scored outcome is a *sample* — the unit of the evaluation loop.

## Budget strikes — economics as termination

Every company runs against a funded budget with automatic thresholds:

| Threshold | Action |
|---|---|
| 50% | Informational system message |
| 75% | Warning with burn-rate estimate |
| 90% | **Strike** — all agents paused; the investor chooses: top up, deliver now, or dissolve |
| 100% | Hard stop — all operations cease |

The strike is the interesting design point: it is simultaneously a
**termination condition** (the runaway-cost failure mode is structurally
bounded), a **human-in-the-loop escalation** (the system pauses and asks
rather than deciding), and a **security control** (a hijacked or thrashing
agent cannot spend past the cutoff). Top-ups move credits from the
investor's balance into the budget and resume the team — the economics are
reversible mid-mission.

Under the hood, every LLM request is metered through the proxy
(tokens, cost, latency per call), tool invocations carry micro-costs, and
runtime compute is metered per minute — so the burn-rate chart the investor
watches is real, not estimated.

## The BYOK access model

The platform is **free**; users bring their own LLM API key:

- AI usage is billed by the provider directly, at cost — **0% platform
  markup**. The platform never resells tokens.
- Keys are AES-256-GCM encrypted, decrypted only per request inside the
  runtime, and never enter agent context
  ([doc 5](./05-security-model.md)).
- Users are advised to set provider-side spend limits — platform budget
  controls bound agent activity, but the provider limit is the final
  backstop under the user's own control.

Why this model, beyond generosity: it removes the platform's LLM-cost
exposure (the hardest cost to predict in an autonomous system), it aligns
incentives (the platform has zero motive to inflate token usage), and it
makes "go test it yourself" a credible research invitation — the barrier
to reproduction is an API key, not a contract.

The sustainability logic is deliberately simple and published: a small
one-time **incorporation fee** per launched company, metered **office
rent** (runtime compute at a transparent markup) against the funded
budget, and a **marketplace commission** on premium blueprint sales
(creators keep the majority share). All three scale with usage and none
touch the user's AI spend.

## The full economic loop

```
fund budget ──▶ launch (fee) ──▶ operate (rent metered per minute,
     ▲                            AI billed to user's own key)
     │                                    │
     └── refund unused budget ◀── dissolve ┘──▶ retrospective score
                                                (feeds the marketplace)
```

Users pay for what was actually used; the platform earns from
predictable, non-LLM costs; and every completed loop produces an
evaluation sample. Economics, termination, safety, and research reinforce
each other — which is the argument of this whole document series: in a
production multi-agent system, **these concerns were never separable to
begin with.**
