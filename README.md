# FlashCo — Technical Documentation

> The open architecture and implementation documentation of
> **[FlashCo](https://flashco.app)** — a reference implementation for
> production multi-agent systems.

FlashCo runs **temporary AI companies**: you launch a team of 2–8 AI agents
from a reusable *blueprint*, they work a mission for hours or days — talking,
delegating, producing real deliverables in your connected tools — and then
the company dissolves itself, refunds unused budget, and scores its own
performance.

The platform is free to use with your own LLM API key (billed by your
provider at cost, 0% markup). Its purpose is research: to design, run, and
keep improving reliable multi-agent systems **in the open**. The source code
is not public; this documentation is — every architectural decision,
pattern, and trade-off is described here so the design can be studied,
critiqued, and reproduced.

**Try it:** [flashco.app](https://flashco.app) · **Interactive pattern
explorer:** [flashco.app/#patterns](https://flashco.app/#patterns)

> **A note on terminology.** These documents use the architecture's
> internal vocabulary: a ***blueprint*** is the durable, versioned design
> of an agent team, and a ***company*** is one run of it. The product UI
> now uses friendlier words for the same two objects: the blueprint is
> called a **company** (the lasting entity you design, hire from the
> marketplace, and that accumulates experience), and each run is a
> **mission** (launched, immutable once started, dissolved, and judged).
> Read `blueprint → company` and `company → mission` when moving between
> these docs and the app — the concepts and code are identical.

---

## The document series

| # | Document | What it covers |
|---|----------|----------------|
| 1 | [System Architecture](./01-system-architecture.md) | The three-tier control-plane / agent-runtime split, stateless MCP tool servers, and the event-sourced state mirror |
| 2 | [Blueprint Notation](./02-blueprint-notation.md) | The five-file notation that makes agent teams declarative, forkable, and sellable |
| 3 | [Orchestration Patterns](./03-orchestration-patterns.md) | Hierarchy, PM-as-dispatcher, handoff/agents-as-tools, dependency task graphs, and termination conditions |
| 4 | [Memory & Context Engineering](./04-memory-and-context.md) | Three-tier memory, observation masking, context isolation, prompt caching, and why architecture beats compaction |
| 5 | [Security Model](./05-security-model.md) | Prompt-injection defense, the Rule of Two, three-layer least privilege, and credential isolation |
| 6 | [Evaluation Methodology](./06-evaluation-methodology.md) | Trajectories, the LLM judge, dimensional scoring, judge calibration, and the regression loop |
| 7 | [The Integration Layer (MCP)](./07-integration-layer-mcp.md) | Dual-transport MCP, stateless tool servers, runtime discovery, and the anti-hallucination rule |
| 8 | [Economics & Lifecycle](./08-economics-and-lifecycle.md) | The company state machine, budget strikes as termination, the BYOK model, and deliberate mortality |
| 9 | [Roadmap & Known Gaps](./09-roadmap.md) | What is not built yet — published, not hidden |

## Implementation deep-dives (with code)

The second series goes below the architecture into the running code:
real, annotated snippets from the production runtime and control plane,
with the engineering decisions and the bugs they prevent.

| # | Document | What it implements |
|---|----------|--------------------|
| 10 | [The Agent Execution Loop](./10-implementing-the-agent-loop.md) | The turn loop: two-stage timeouts, tool dispatch, error-as-context, MCP auto-reconnect, history sanitization |
| 11 | [Three-Tier Memory](./11-implementing-three-tier-memory.md) | Token budgets, observation masking, episodic facts, safe summarization cut points, prompt assembly |
| 12 | [The Security Guard](./12-implementing-the-security-guard.md) | Named injection patterns, flag-and-frame wrapping, typed secret redaction, the Rule of Two policy engine |
| 13 | [Schema-Forced Structured Output](./13-implementing-structured-output.md) | Forced tool calls in TS and Python, schemas as semantics, acyclic-by-construction task graphs |
| 14 | [The LLM Judge](./14-implementing-the-llm-judge.md) | Trajectory gathering, judge calibration against known biases, weighted scoring, fire-and-forget operation |
| 15 | [Per-Agent Tool Filtering](./15-implementing-tool-filtering.md) | The three-layer filter, the deny-all sentinel, and keeping two allowlists coherent |
| 16 | [Budget Strikes & Metering](./16-implementing-budget-strikes.md) | One metering gate, the strike ladder, BYOK branching, top-up as a conservation law |
| 17 | [Orchestration Modes](./17-orchestration-modes.md) | Four topologies (hub, relay, auction, panel) behind one strategy seam — and how each run is measured |
| 18 | [Agent Skills & the Blueprint Experience Loop](./18-agent-skills-and-blueprint-experience.md) | Portable SKILL.md expertise per agent, and the Hermes-style loop that writes each run's lessons back into the blueprint |

Each document stands alone; together they describe the complete system —
the design series (1–9) explains *why*, the implementation series (10–18)
shows *how*.

## How this maps to the literature

The architecture is deliberately mapped against the published body of
multi-agent research — orchestration taxonomies, context engineering,
evaluation-driven development, and agentic security (including
*Designing Multi-Agent Systems* by Victor Dibia, whose pattern taxonomy we
use as a coverage checklist). Where FlashCo implements a published pattern,
these docs say so; where it diverges, they explain why; what is missing is
in the [roadmap](./09-roadmap.md).

## Reading order

- **Building your own multi-agent system?** Start with 1 → 3 → 4 → 5, then
  the matching implementation docs 10 → 11 → 12.
- **Evaluating agent platforms?** Start with 2 → 6 → 8, then 14 → 16.
- **Security review?** 5 → 7 → 1, then 12 → 15.
- **Just curious what this is?** [flashco.app](https://flashco.app) first —
  the landing page walks the concepts interactively, then come back here
  for depth.

## Status

This documentation describes the system as deployed at
[flashco.app](https://flashco.app). It evolves with the platform; the
roadmap doc tracks what changes next.

*Maintained by [Kacem Ait Oual](mailto:kacem.aitoual@gmail.com). Feedback
and critique welcome — that is the point of publishing it.*
