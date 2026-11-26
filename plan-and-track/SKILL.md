---
name: plan-and-track
description: "Set and track goals, KPIs, OKRs, V2MOMs, quotas, spend budgets, burn caps, measures, operating plans, plan-vs-actual and custom business views in Economico — always measured from the live books and pipelines."
---

# Plan and track (goals, KPIs, OKRs, V2MOMs & operating plans)

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; fetch recipes via `catalog {action: "recipe", name}`; confirm the target business
before writing). Recording, revising, activating or reading measures, targets and custom views
never posts to a ledger; progress always comes from the books and open pipelines.

## What this skill owns

1. **Define KPIs and measures:** Draft SQL measures over the report schema, test them with
   `measures {action: "try"}` against a real window, and version them with `measures.define`.
2. **Set goals, OKRs, V2MOMs, quotas, spend budgets and operating plans:** Group company, pipeline
   and per-person targets under a `plan` label (`targets.define` proposed-first, confirm numbers
   with the founder, then `targets.activate`), modeling spend budgets and burn ceilings as
   `comparison: "at_most"` targets over expense accounts, payroll or net burn. Check pace, cap
   usage and open-pipeline coverage (`targets {action: "read"}`), and revise mid-period with
   `expectedVersion` and a `reason`. Read [targets and measures](references/targets.md).
3. **Build and share live business views:** Publish custom reports (`report_format` +
   `report_definitions.publish`) for plan-vs-actual, weighted pipelines, collections, renewals,
   spend and customer economics, and share live or frozen links via `shared_views.create`. Read
   [business views](references/business-views.md).

## References and recipes

- [Targets and measures](references/targets.md) — measure SQL rules, proposed-first convention,
  dynamic budgets, pace/coverage formulas, plan-vs-actual report
- [Starter measure recipes](references/measure-recipes.md) — 22 tested SQL measure definitions
  (pipeline, bookings, MRR/ARR, burn, runway, headcount, quotas, dynamic budgets)
- [Business views](references/business-views.md) — weighted pipeline report, founder data platform
  views, cross-function map and business shapes
- Shared foundations in `economico`: [pipelines](../economico/references/pipelines.md),
  [asking the founder](../economico/references/asking-the-founder.md),
  [verify](../economico/references/verify.md), [show the founder](../economico/references/show-the-founder.md)
- Fetch recipes and report templates via `catalog {action: "recipe", name}`:
  `targets-sales-quarter-quotas`, `targets-operating-plan`, `targets-hiring-plan`,
  `targets-fundraise`, `targets-dynamic-budgets`, `plan-vs-actual`, `weighted-pipeline`
