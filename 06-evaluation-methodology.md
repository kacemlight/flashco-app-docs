# 6. Evaluation Methodology

*"You cannot optimize what you cannot measure"* is the most-repeated and
least-implemented advice in agent engineering. FlashCo's approach:
**every production run is an evaluation.** No separate benchmark harness,
no synthetic test suite as the primary signal — the companies users
actually launch are the dataset.

## The trajectory

A dissolved company leaves a complete trajectory, captured as a side effect
of the event-mirror architecture ([doc 1](./01-system-architecture.md)):

- **Messages** — the full conversational flow: agent ↔ agent, agent ↔
  investor, system events;
- **Actions** — every tool call with its parameters, outcome, and micro-cost;
- **Tasks** — the planned graph vs. what reached done/blocked/abandoned;
- **Outcomes** — the deliverable files actually produced;
- **Metadata** — per-request tokens, cost, latency; budget consumed vs.
  funded.

Nothing extra is instrumented for evaluation; observability *is* the
evaluation substrate.

## The judge

At dissolution, an LLM judge (fire-and-forget — an evaluation failure can
never block a company's shutdown) receives the assembled trajectory:
mission, task board, deliverable listing, a recency-bounded message sample,
and cost aggregates. It scores through a **schema-forced tool call** —
structured output by construction, no free-text parsing — across five
weighted dimensions:

| Dimension | Weight | What it measures |
|---|---|---|
| Deliverable quality | 35% | Coverage and quality of outputs vs. the stated mission |
| Task completion | 25% | Done/blocked ratio, judged in budget-and-deadline context |
| Collaboration | 15% | Delegation quality, handoffs, escalation, duplicated or conflicting work |
| Efficiency | 15% | Value per token; budget utilization — never absolute size |
| Communication | 10% | Clarity, timeliness, and honesty of investor updates (including about problems) |

Each dimension returns a score **with reasoning**, plus overall strengths,
weaknesses, and a summary. The weighted overall, the per-dimension detail,
and a trajectory-statistics snapshot are stored; a scorecard message is
posted into the company's own chat, where the investor already looks.

## Judge calibration — the part everyone skips

LLM judges have documented biases, and evaluation itself requires
debugging. Two calibrations are baked into the judge prompt:

- **No verbosity penalty.** Multi-agent systems narrate their coordination;
  a naive judge reads that as noise and punishes exactly the transparency
  that makes the system auditable. The judge is explicitly instructed to
  score outcomes and collaboration quality, not brevity. (The reference
  literature documents scores jumping ~2.5 points after fixing precisely
  this bias.)
- **Context-aware completion.** Incomplete task boards are judged against
  *why* — a company stopped by its budget strike is not the same failure as
  one that lost the plot.

The judge model's identity is recorded with every evaluation, so scores
remain comparable across judge upgrades — and old companies can be
re-judged on demand for calibration studies.

## The two feedback loops

**Loop 1 — platform regression detection.** Any change to prompts,
orchestration, or memory policy can be checked against before/after score
distributions for the same blueprint. The trajectory statistics
(tokens, cost, tasks completed) give matching efficiency curves. This is
evaluation-driven development applied to a live system: the measurement
infrastructure existed *before* the optimization program it enables.

**Loop 2 — marketplace quality signal.** Scores attach to the blueprint a
company was launched from. Creators see how their designs perform across
real missions (not just star ratings), and the marketplace can surface
designs that measurably work. Evaluation becomes the connective tissue
between the research goal ("improve multi-agent design") and the product
("a library of designs that provably improve").

## Design choices worth defending

- **Reference-free, rubric-based scoring.** Missions are open-ended; there
  is no gold answer to compare against. The rubric encodes what an expert
  reviewer would check, which is the honest version of evaluation for
  generative work.
- **Platform-funded judging.** Evaluation is a platform quality feature —
  it runs on the platform's model budget, not the user's key, and silently
  skips if unconfigured. Users are never billed for being measured.
- **One judged sample per run, at the natural boundary.** Dissolution is
  the moment the full trajectory exists and the company's story is
  complete. Mid-flight step evaluation is a separate (planned) mechanism —
  see the [roadmap](./09-roadmap.md) — with a different purpose: recovery,
  not scoring.

## Honest limitations

- A single judge call per company means noisy per-run scores; the signal is
  in distributions per blueprint, not individual numbers.
- No human-expert validation set yet — judge scores have not been
  correlated against blinded human review at scale.
- Process metrics (tool-selection correctness, argument validity) are
  captured in the trajectory but not yet scored as first-class dimensions.
- Behavioral-alignment evaluation (sycophancy, drift) is absent — tracked
  on the roadmap.
