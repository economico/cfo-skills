---
name: investor-update
description: Build and publish this month's investor update from the books in Economico as a frozen report link or Markdown draft.
---

# Investor update

Follow [economico](../economico/SKILL.md) for ground rules (confirm the business slug before writing;
money is minor units; never invent figures).

A monthly update for the founder's investors, built as a custom report: a page Economico draws
from the books, with the founder's words in it, sent as a link that shows one month and never
changes. The books supply the figures, charts and tables; the founder supplies the story. Write
it for the month that just closed unless the founder names another.

The report is published once and then again each month as a new version, with that month's
highlights, lowlights and asks. When this connection may not publish, write the
[Markdown update](#when-the-connection-may-not-publish) instead.

## Build the report

1. `report_format {action: "guide"}`. It is the reference for the format, and its
   `investor_update` example is the definition to start from: headline figures, charts over the
   months the books have, top-ten tables with the change from the month before, a text block
   for the story, and one `month` parameter. Do not write a definition from memory.
2. `reports {action: "definitions"}`. If the business already has an investor update, read it
   with `report_format {action: "source", id}` and keep its `id`: this month is a new version
   of that report, not a second report.
3. Fit the definition to this business. Drop a block the books cannot answer (the usage blocks,
   for a business that meters nothing) and tell the founder which. Before changing a query,
   read `report_sql {action: "schema"}`.
4. `report_format {action: "check", definition}` until it passes. A refusal names the block at
   fault.
5. `reports {action: "preview", definition, params}` with `params` naming the month as
   `YYYY-MM`. Read every block: one that answers `error` is broken, and a chart with no rows is
   drawn as "No data". Compare cash, net burn and recurring revenue with `reports run
   saas_metrics` for the same `year` and `month`: they are the same figures, so a difference
   means the wrong month. Inspect `monthly_metrics.excluded_token_units` and the report's
   `excluded_token_units`/`missing_rates`; name excluded holdings beside the figures. Par-valued
   stablecoins in the reporting currency are included already, so do not add them twice.

When the books for the month are incomplete (receipts not yet recorded, a Stripe payout
missing), say which, and offer to record them first: an update built on partial books understates
burn.

The headline figures are in the books' functional currency only. For mixed-currency books, read
[native and indicative currency totals](../economico/references/indicative-currency.md) and state
the other currencies' native amounts in the text block, with their cutoff and rate provenance;
never add them to the report's figures or show an invented month-on-month change.

## Operating plan targets and dynamic budgets in investor updates and board packs

When the business has active operating-plan targets or dynamic budgets:

1. List them with `targets {action: "list"}` (or filter by `label_key: "plan"` for the active
   operating plan).
2. Read each target for the reporting period:
   - Pass `targets {action: "read", id, current_only: true}` when reporting the current or latest
     started period; omit `current_only` when reporting an earlier closed month so that month is
     evaluated (and follow `next_cursor` with the same `version` if the month is more than 12
     started periods back).
   - When comparing against the original board-approved plan after a mid-year revision, confirm the
     adopted version with `targets {action: "versions", id}` and also call
     `targets {action: "read", id, version: <board>}` (always compare revisions through
     `targets read` on both versions; see [targets](../plan-and-track/references/targets.md#operating-plan-recipe)).
3. Include an **Operating plan & dynamic budgets** summary beside the financials following the
   formatting and gap-sign rules in [targets](../plan-and-track/references/targets.md#one-table-recipe):
   - **In a custom report:** embed a native `type: "targets"` block (`plan` literal or `:param`, or
     explicit `target_ids`, with optional `as_of` or single-target `version`) when showing the
     current period — it evaluates live on preview and freezes cleanly into `shared_views`
     snapshots with `%` formatting, `Underlying` context, open-pipeline estimates, and over-budget
     badges. When reporting an earlier closed month or a side-by-side board baseline comparison,
     add a `text` summary table populated from `targets read` instead. Avoid embedding
     `plan-vs-actual` (`catalog {action: "recipe", name: "plan-vs-actual"}`) SQL blocks inside
     `investor_update`: `plan-vs-actual` is built for standalone live USD-target reports (it gates
     on `date('now')` instead of `:month`, reads only the latest target version from
     `target_period_values`, and expects currency minor units rather than `percent` or scaled
     `number` dynamic budgets).
   - **In a Markdown update or board pack:** place the same compact table right after the headline
     metrics.
4. Keep **stated values**, **recorded actuals**, **underlying `context` figures** (such as real
   travel dollars and headcount, or R&D token spend and Net New Revenue behind a rate cap), and
   **open-pipeline estimates** (`estimate`) in clearly separated columns or lines. **Open-pipeline
   estimates and projected forward P&Ls are never added into recorded actuals**; if a board pack
   includes a forward projection from [targets](../plan-and-track/references/targets.md#what-if-projected-pl-from-targets-and-dynamic-budgets),
   label it prominently as **Illustrative projection · unposted** with every assumption stated.

## Ask the founder for the story

In one round ([asking the founder](../economico/references/asking-the-founder.md)): the month's
highlights, the lowlights, and the asks (intros, hires, advice). Offer choices drawn from the books
where they suggest one (a new customer, a large contract, an overdue invoice), and leave room for
their own words.

## Publish and send

1. Put the founder's words in the text block under Highlights, Lowlights and Asks, short and in
   their voice. Check and preview again, and show the founder the preview: the host draws it
   where it renders MCP Apps; elsewhere repeat the figures as a table
   ([show the founder](../economico/references/show-the-founder.md)).
2. When the founder agrees, publish with `report_definitions.publish` through `commands`: the
   `definition`, and the report's `id` every month after the first. The result has the `id` and
   the new `version`. Publishing records an event in the books; say which business first.
3. Offer to pin it: `report_definitions.pin` with the `id` puts the report in every member's
   workspace sidebar, and `overview: true` also draws it on the overview. A pin follows the
   latest version.
4. Offer the link to send. With the founder's agreement, `shared_views.create` with a
   `description`, `definitions: [{id, version, params: {month: "YYYY-MM"}}]` and
   `mode: "frozen"`. A frozen link captures that month's figures as they stand and never
   changes, which is what an update should do (a `live` link follows the books; see
   [verify](../economico/references/verify.md#an-answer-the-founder-wants-to-keep)); anyone holding
   it can open it without an account, at `https://<host>/e/<slug>/share/<token>`. Sharing needs a
   paid plan: on `plan_required`, say so and give the founder the Markdown update instead.

Economico does not send the update. Give the founder the link and one line on the month to send
with it.

## When the connection may not publish

You can tell from the first refusal: `report_format {action: "guide"}` answers `forbidden` (this
connection may not query the books), or `report_definitions.publish` is refused or missing from
`catalog {action: "list"}` (it may not manage reports). Do not work around it. Write the update
as Markdown the founder can paste into an email, from the built-in reports:

| Section | Read | Use |
|---|---|---|
| Recurring revenue | `reports run saas_metrics` with `year` and `month`, for this month and the one before | MRR and ARR, and their change on last month |
| Revenue and spend | `reports run income_statement` with `from` the first of the month and `until` the next month's first instant as a UTC timestamp (`2026-10-01T00:00:00Z` for September) | Revenue, cost of revenue, gross margin, operating expenses, net income or loss |
| Cash and runway | `reports run default_alive` with `period` `YYYY-MM`; for a token it names in `excluded_token_units`, `ledgers {action: "balances", ledger_id, as_of}` on that token's ledger at the same month-end cutoff (`holdings` in `accounts overview` is today's, not the month's) | Cash at month end, net burn, months of runway, stablecoins at par in the reporting currency included; list a token named in `excluded_token_units` beside cash, wallet by wallet, as left out of cash and burn (an `indicative` block already counts a stablecoin pegged to another currency) |
| Money owed to the company | `reports run aging` with `direction` `receivable` and `as_of` the next month's first instant as a UTC timestamp (`2026-10-01T00:00:00Z` for September) | Anything overdue worth a line |
| Operating plan & dynamic budgets *(when `report_sql` read access is granted)* | `targets {action: "list"}` and `targets {action: "read", id, current_only: true}` (omit `current_only` and follow `next_cursor` for an earlier closed month; add `version: <board>` for board-baseline comparison) | Stated targets and dynamic budget rates vs. actuals, with supporting `period.context[]` dollars/drivers and open-pipeline estimates kept separate from actuals; omit this section rather than retrying when `report_format` or `targets` returns `forbidden` |

Scale every amount once by the ledger's `decimals`, as [verify](../economico/references/verify.md)
says. Compare with last month by reading last month the same way; never estimate a figure the
reports did not return. If a combined view of mixed currencies is unavailable, retain native
amounts and name the missing pairs and days; do not show a zero or a partial total.

Short, in the founder's voice:

1. One line on the month.
2. Metrics: MRR and ARR with the change, revenue, net burn, cash, runway in months (plus an Operating plan & dynamic budgets summary when active targets exist, keeping stated values, actuals, `context` figures, and open-pipeline estimates cleanly separated).
3. Highlights, lowlights, asks.

Save it as `investor-updates/<YYYY-MM>.md` in the founder's repository when there is one. It is a
draft: Economico does not send it, and nothing is recorded on the books.
