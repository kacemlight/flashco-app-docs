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

Each document stands alone; together they describe the complete system.

## How this maps to the literature

The architecture is deliberately mapped against the published body of
multi-agent research — orchestration taxonomies, context engineering,
evaluation-driven development, and agentic security (including
*Designing Multi-Agent Systems* by Victor Dibia, whose pattern taxonomy we
use as a coverage checklist). Where FlashCo implements a published pattern,
these docs say so; where it diverges, they explain why; what is missing is
in the [roadmap](./09-roadmap.md).

## Reading order

- **Building your own multi-agent system?** Start with 1 → 3 → 4 → 5.
- **Evaluating agent platforms?** Start with 2 → 6 → 8.
- **Security review?** 5 → 7 → 1.
- **Just curious what this is?** [flashco.app](https://flashco.app) first —
  the landing page walks the concepts interactively, then come back here
  for depth.

## Status

This documentation describes the system as deployed at
[flashco.app](https://flashco.app). It evolves with the platform; the
roadmap doc tracks what changes next.

*Maintained by [Kacem Ait Oual](mailto:kacem.aitoual@gmail.com). Feedback
and critique welcome — that is the point of publishing it.*
