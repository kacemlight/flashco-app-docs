# 14. Implementing the LLM Judge

*Implementation deep-dive. Design context: [doc 6 — Evaluation
Methodology](./06-evaluation-methodology.md). Snippets from the production
control plane (Python), lightly trimmed.*

The judge turns every dissolved company into an evaluation sample. Its
implementation is three functions — gather, prompt, score — and a set of
operational rules that matter more than any of the three.

## Rule zero: evaluation never breaks the product

```python
async def run_company_retrospective(company_id: str) -> dict | None:
    """
    Judge a company's full trajectory and persist the evaluation.
    Never raises: returns the stored evaluation, or None when skipped
    (no API key) or failed (logged).
    """
    settings = get_settings()
    if not settings.anthropic_api_key:
        logger.info("Skipping retrospective for %s: no judge key", company_id)
        return None
    try:
        ...
    except Exception:
        logger.exception("Retrospective evaluation failed for %s", company_id)
        return None
```

And at the call site, dissolution schedules it fire-and-forget:

```python
# In dissolve_company(), after deliverables are compiled and refunds issued:
try:
    asyncio.create_task(run_company_retrospective(company_id))
except Exception:
    logger.exception("Failed to schedule retrospective for %s", company_id)
```

A user's dissolution — and their refund — must never wait on, or fail
because of, a scoring call. Measurement is a parasite on the product, never
a dependency.

## Gathering the trajectory

The judge reads what the event-mirror architecture
([doc 1](./01-system-architecture.md)) already recorded — nothing is
instrumented specially for evaluation:

```python
def _gather_trajectory(company_id: str) -> dict:
    company = ...   # mission, mode, status, budget fields
    tasks   = ...   # full task board: title, status, priority

    # Last 60 messages: collaboration texture without context blowout
    messages = (db.table("messages")
        .select("type, content, created_at")
        .eq("company_id", company_id)
        .order("created_at", desc=True).limit(60)
        .execute()).data or []
    messages.reverse()

    logs = ...          # per-request tokens, cost, latency
    deliverables = ...  # file listing from storage (best-effort)

    stats = {
        "tasks_total": len(tasks),
        "tasks_done": sum(1 for t in tasks if t["status"] == "done"),
        "llm_requests": len(logs),
        "tokens_input":  sum(int(l.get("tokens_input") or 0) for l in logs),
        "tokens_output": sum(int(l.get("tokens_output") or 0) for l in logs),
        "cost_total": round(sum(float(l.get("cost_total") or 0) for l in logs), 4),
        "deliverables_count": len(deliverables),
        "budget_total": float(company.get("budget_total") or 0),
        ...
    }
    return {"company": company, "tasks": tasks, "messages": messages,
            "deliverables": deliverables, "stats": stats}
```

Choices worth noting: the message sample is *recency-bounded* (the judge
sees how the mission ended, which is where collaboration quality shows);
storage listing is best-effort (a storage hiccup shouldn't kill scoring);
and the raw aggregates ship alongside the scores as `trajectory_stats`, so
later analysis can correlate judgments with hard numbers.

## The calibrated prompt

Judge bias is a documented failure mode, and the two most important lines
of the prompt exist because of it:

```python
prompt = f"""You are evaluating the complete trajectory of an AI multi-agent
company that has finished its mission. Score each dimension 0-10 with
reasoning, then submit via the submit_evaluation tool.

CALIBRATION RULES:
- Do NOT penalize verbosity or process transparency: agents narrating their
  coordination is a feature of this system, not noise. Judge outcomes and
  collaboration quality, not brevity.
- task_completion: {stats['tasks_done']}/{stats['tasks_total']} tasks reached
  done. Blocked or abandoned tasks reduce the score; consider whether
  incompleteness was budget/deadline-driven.
- efficiency: {stats['llm_requests']} LLM requests,
  {stats['tokens_input']:,} input + {stats['tokens_output']:,} output tokens,
  {stats['cost_total']} credits consumed of {stats['budget_total']} budget.
  Judge value-per-token, not absolute size.
...
"""
```

- **The anti-verbosity rule** counters the strongest known bias: LLM
  judges read multi-agent coordination narration as bloat and punish
  exactly the transparency that makes the system auditable. (The reference
  literature documents scores jumping ~2.5 points after fixing this.)
- **Hard numbers inside the rubric.** The judge isn't asked to *estimate*
  completion or cost — the true counts are printed into the instructions,
  so its job is interpretation, not arithmetic. LLMs are bad accountants
  and decent adjudicators; the prompt assigns each role accordingly.
- **Context-aware incompleteness.** A company stopped by its budget strike
  is a different failure from one that lost the plot; the rubric says so.

## Scoring, weighting, and the paper trail

The response comes back through a forced tool call (schema in
[doc 13](./13-implementing-structured-output.md)) — every dimension
guaranteed present, every score an integer 0–10, every score with
reasoning. The weighted overall is computed in code, not by the model:

```python
overall = round(sum(
    DIMENSION_WEIGHTS[dim] * float(evaluation[dim]["score"])
    for dim in DIMENSION_WEIGHTS
), 2)

row = {
    "company_id": company_id,
    "template_id": trajectory["company"].get("template_id"),  # → marketplace loop
    "overall": overall,
    "scores": {dim: evaluation[dim] for dim in DIMENSION_WEIGHTS},
    "reasoning": ...,             # strengths / weaknesses / summary narrative
    "trajectory_stats": trajectory["stats"],
    "model": JUDGE_MODEL,         # recorded for cross-upgrade comparability
}
db.table("company_evaluations").insert(row).execute()
```

Then the score goes where the user already looks — their company's chat:

```python
db.table("messages").insert({
    "company_id": company_id,
    "content": f"**Company Retrospective — overall {overall}/10**\n\n"
               + per_dimension_lines + summary,
    "type": "system",
}).execute()
```

No new UI was built for evaluation: the existing realtime pipeline
delivers the scorecard. Access control mirrors the marketplace loop —
investors read their own companies' evaluations, and **blueprint creators
read evaluations of companies launched from their designs** (row-level
security), which is how scores become a quality signal on the
marketplace.

## Why the judge model is recorded per row

Scores are only comparable under the same judge. Recording `model` with
every evaluation means: upgrade the judge freely, re-judge old companies
through the on-demand endpoint, and compare distributions across judge
versions before trusting mixed data. Evaluation infrastructure that can't
evolve its own judge is a dead end; this one column keeps it alive.

## Operational endpoints

- **Automatic** — scheduled at dissolution (above).
- **`POST /companies/{id}/evaluate`** — on-demand re-judging: backfills,
  calibration studies, judge-upgrade comparisons.
- **`GET /companies/{id}/evaluation`** — latest result.

## The limitation the implementation is honest about

One judge call per company means noisy individual scores — the signal
lives in *distributions per blueprint*, not single numbers. The
implementation leans into this rather than hiding it: per-dimension
reasoning makes individual scores inspectable, `trajectory_stats` makes
them correlatable, and the roadmap ([doc 9](./09-roadmap.md)) adds
human-validation sets and process-level metrics as the next rigor tier.
