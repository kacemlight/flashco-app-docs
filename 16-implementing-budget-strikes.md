# 16. Implementing Budget Strikes & Metering

*Implementation deep-dive. Design context: [doc 8 — Economics &
Lifecycle](./08-economics-and-lifecycle.md). Snippets from the production
control plane (Python), lightly trimmed.*

"Agents that don't know when to stop" is the canonical production failure
of autonomous systems. FlashCo's answer makes *economics* the termination
authority: every cost flows through one metering function, and that
function is also where the strike thresholds live. You cannot spend
without being measured, and you cannot be measured without being checked
against the budget.

## One gate for every cost

```python
async def meter_usage(
    company_id: str,
    cost: Decimal,
    cost_category: str = "ai_model",   # "ai_model" | "compute" | tool costs
    ...
) -> dict:
    company = ...  # budget_total, budget_consumed, ai_billing_mode, ...
    billing_mode = company.get("ai_billing_mode", "platform")

    # BYOK: AI costs are tracked for display but NOT deducted —
    # the user pays their model provider directly.
    should_deduct = True
    if billing_mode == "byok" and cost_category == "ai_model":
        should_deduct = False

    if should_deduct and cost > 0:
        await deduct_credits(user_id, cost, ..., cost_category=cost_category)

    # Per-category consumption on the company row
    if cost_category == "compute":
        update["compute_consumed"] = float(old_compute + cost)
        update["budget_consumed"] = float(old_consumed + cost)
    elif cost_category == "ai_model":
        if billing_mode == "byok":
            update["ai_consumed_estimated"] = float(old_est + cost)  # display only
        else:
            update["ai_consumed"] = float(old_ai + cost)
            update["budget_consumed"] = float(old_consumed + cost)

    # Per-minute bucket for the live burn-rate chart
    db.table("credit_consumption_log").insert({
        "company_id": company_id,
        "minute_bucket": now.replace(second=0, microsecond=0).isoformat(),
        ("compute_cost" if cost_category == "compute" else "ai_cost"): float(cost),
    }).execute()

    # EVERY metering call checks the thresholds — no separate watchdog
    await check_thresholds(company_id, actual_consumed, float(budget_total))
```

Three properties fall out of this shape:

- **BYOK is one branch, not a parallel system.** Under
  bring-your-own-key, AI cost is *estimated for display* (the investor
  still sees what their missions cost) but never deducted — the provider
  bills them directly. Office rent (compute) deducts in both modes.
- **The burn-rate chart is real.** Per-minute buckets are written at
  metering time; the dashboard sparkline is a projection of actual charges,
  not an estimate.
- **Threshold checking rides the metering path.** There is no cron
  watching budgets; the check runs exactly when consumption changes, which
  is the only time it can change state.

## The strike ladder

```python
async def check_thresholds(company_id, budget_consumed, budget_total):
    pct = budget_consumed / budget_total

    if pct >= 1.0:
        # 100% — hard stop (safety net)
        db.table("companies").update({"status": "draft"}).eq("id", company_id).execute()
        db.table("agents").update({"status": "paused"}) \
          .eq("company_id", company_id) \
          .in_("status", ["active", "working", "idle"]).execute()
        post_system_message("**BUDGET EXHAUSTED** — All operations stopped.")

    elif pct >= 0.9:
        # 90% — STRIKE: pause everything, hand the decision to the human
        pause_all_agents(...)
        post_system_message(
            f"**STRIKE — Budget at 90%+**\n\n"
            f"All agents paused. Remaining: {remaining:.2f} credits\n\n"
            f"Options:\n"
            f"- **Top Up** — add credits to continue\n"
            f"- **Deliver Now** — compile current work and close\n"
            f"- **Dissolve** — stop and refund remaining credits"
        )

    elif pct >= 0.75:
        post_system_message(f"**Budget Alert — 75% consumed** … "
                            f"(~{remaining_hours:.0f}h at current rate)")

    elif pct >= 0.5:
        post_system_message("Budget update: 50% remaining. All systems nominal.")
```

The 90% strike is the design's center of gravity. It is simultaneously:

- a **termination condition** — runaway burn is structurally bounded;
- a **human-in-the-loop escalation** — the system pauses and *presents
  options* rather than deciding; the message is the UI;
- a **security control** — a hijacked or thrashing agent hits the same
  wall as an enthusiastic one.

Note the message includes the projected runway ("~Nh at current rate") at
75% — computed from actual burn, giving the investor time to decide
*before* the strike, not after.

## Top-up: credits move, they don't appear

The strike's recovery path. The subtle invariant: a top-up **moves**
credits from the investor's wallet into the company budget — it must never
mint them:

```python
async def topup_company(company_id: str, amount: Decimal) -> dict:
    company = ...  # investor_id, budget_total, status

    # Move credits from the investor's balance into the company budget.
    # Raises on insufficient balance (caller returns 4xx).
    new_balance = await deduct_credits(
        user_id, amount, f"Top-up for company ({float(amount):.2f} credits)", company_id,
    )

    update = {"budget_total": float(budget_total + amount)}
    if company["status"] == "draft":          # struck → resume
        update["status"] = "active"
        resume_paused_agents(company_id)
    db.table("companies").update(update).eq("id", company_id).execute()

    post_system_message(f"**Funding received** — {float(amount):.2f} credits added.")
```

Worth admitting in public documentation: an earlier version of this
function called `deposit_credits` instead of `deduct_credits` — topping up
*added* to the investor's wallet **and** raised the budget, a money
printer found during a systematic billing audit. The lesson generalizes:
in agent platforms, every credit mutation should be writable as a
*transfer with a conservation law* ("wallet + budgets = constant except at
Stripe and refunds"), and audited against it.

## Dissolution closes the loop

At dissolution the ledger settles: total consumption is computed from the
per-request logs (not from counters that could drift), the refund is
simply the unspent budget, and the cost breakdown is posted to the
investor:

```python
total_consumed = sum(Decimal(str(r["cost_total"])) for r in request_logs)
refund_amount = max(budget_total - total_consumed, Decimal("0"))
# → refund transaction + final system message with the full breakdown
```

Users pay for what was used; everything else comes back. Combined with the
retrospective that fires at the same moment
([doc 14](./14-implementing-the-llm-judge.md)), every company ends with
two artifacts: a settled bill and a scored trajectory. Economics and
evaluation, closing together — which is the thesis of this whole series:
in production multi-agent systems, the "boring" mechanisms are the
architecture.
