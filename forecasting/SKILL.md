---
name: forecasting
description: >
  Project Economico's future state — cash, runway, MRR/ARR, and committed
  inflows/outflows — by running what-ifs and forward-dated draft entries inside
  disposable planning scenarios, never touching the real books: reports show base
  + scenario and the real ledger stays provably unchanged. Use when someone wants
  to see the future rather than record the present. Triggers on: "forecast",
  "projection", "what if we raise prices", "model a new deal", "what if this
  customer churns", "runway forecast", "cash flow forecast", "expected inflows",
  "committed run-rate", "draft journal", "scheduled entries", "future invoices and
  bills", "scenario planning", "what-if", "project MRR/ARR", "when do we run out
  of cash". For read-only analysis of what already happened use financial-analyst;
  to record real transactions use the money-loop skills (invoicing,
  expense-tracking). Assumes Economico is connected (setup-economico).
---

# Forecasting

You build **forward-looking** financial projections off Economico's real ledger.
Where `financial-analyst` reads what *happened*, you model what *will* or *might*
happen — committed cash flow from contracts, runway, and what-ifs like a price
change, a new deal, or a churned customer.

You do this by **writing into a planning scenario** — a named, additive overlay
on the real ledger. Reports run under the scenario show `base ∪ scenario`; the
real books are never modified by scenario activity, and the whole overlay is
deletable in one cascade. The scenario *is* your isolation boundary and your
"draft" mechanism: forward-dated entries you post inside it project the timeline
without ever becoming real.

**Always forecast through a scenario — never from arithmetic in your head.** This
holds even for a plain "just tell me the runway" ask with no explicit what-if:
the user asking you to *project the numbers forward* is the trigger. Run the full
loop every time — open a scenario, post the projection entries into it, then read
the numbers back under the scenario. Every figure you report must come from an
Economico report run with the `scenario` parameter (`base ∪ scenario`), not from
mental math on the baseline. Concretely: `create_scenario` is your **first write,
right after the baseline read and before you compute anything** — if you have not
created a scenario yet, you are not ready to answer. Eyeballing a runway off the
baseline instead of projecting into a scenario is the most common way to get this
wrong; do not do it.

## Iron rule — never forecast on the real ledger

Every projection entry **must** carry a `scenario` (MCP `scenario` parameter /
CLI global `--scenario <name>`). Posting a hypothetical invoice, bill, or journal
without a scenario writes it to the real books — that is the one thing this skill
must never do. If you are not certain a write is scoped to a scenario, stop and
re-scope it. Reads of the real ledger (to establish the baseline) are fine and
expected.

Safety the platform already gives you: **sending an invoice inside a scenario
posts its AR/revenue journal but never delivers** (no email, no Flow) — a
hypothetical invoice can't reach a real customer. Lean on that, but still prefer
posting forward entries directly over "sending" unless you're modeling collection
timing.

## The scenario lifecycle

| Step | MCP tool | CLI |
|---|---|---|
| Create a fresh overlay | `create_scenario(name, description?)` | `economico scenarios create <name> --description …` |
| List existing overlays | `list_scenarios()` | `economico scenarios list` |
| Branch a what-if to vary it | `clone_scenario(source, name, description?)` | `economico scenarios clone <source> <name>` |
| Re-run from scratch (keep the name) | `reset_scenario(name)` | `economico scenarios reset <name>` |
| Tear it all down | `delete_scenario(name)` | `economico scenarios delete <name>` |

Every business is seeded a `test` sandbox at signup — fine for quick throwaway
math. For anything you'll compare or revisit, create a purpose-named scenario
(`price-hike-q3`, `lose-acme`, `base-case-fy26`). **Branch with `clone_scenario`**
to compare variants from the same starting point (e.g. clone `base-case` into
`bull` and `bear`).

## Workflow

1. **Baseline (real ledger, read-only).** Pull `get_balance_sheet`,
   `get_income_statement`, and `summarize_revenue` for orientation, and
   `list_contracts` / `list_obligations` for the **committed run-rate** — recurring
   obligations are the forward inflows (customer) and outflows (vendor) you're
   already locked into before any invoice/bill exists. This is the spine every
   projection builds on. Note the currency; never mix currencies.
2. **Open the scenario — always, and before any projection.** `create_scenario`
   (or `clone_scenario` from a base case), named for the question. This step is
   mandatory even for a quick runway read; it is the only place your forward
   entries may live and the only way the reports in step 4 can show the projected
   numbers. Never skip from the baseline (step 1) straight to computing — if you
   have not called `create_scenario`, you are not ready for step 3.
3. **Project forward, dated.** Post the future entries *into the scenario*, each
   `external_timestamp` set to its expected book date so they land on the timeline
   (see [`references/playbook.md`](references/playbook.md) for the derivation and
   the draft-entry table format):
   - **Committed inflows** — for each recurring customer obligation, a future AR
     entry per period (`Dr 1120 Accounts Receivable / Cr revenue`).
   - **Committed outflows** — for each recurring vendor obligation, a future AP
     entry per period (`Dr expense/COGS / Cr 2110 Accounts Payable`).
   - **The what-if** — the change you're testing: price up/down (adjust amounts),
     a new deal (add party + contract + obligation + its future invoices), churn
     (stop a customer's future entries), a hire or one-off cost (forward expense).
4. **Report under the scenario.** Re-run `get_balance_sheet` /
   `get_income_statement` / `summarize_revenue` **with the `scenario` parameter** —
   they now return `base ∪ scenario`. Compute ending cash, runway, and the
   MRR/ARR / margin delta versus the baseline.
5. **Compare & present.** Lead with the answer ("At +20% list price, FY26 ending
   cash is $310k and runway extends from 9 to 14 months"), then the projected
   statements, then the assumptions and the scenario name so it's reproducible.
6. **Tear down or keep.** `delete_scenario` throwaways; keep named base/bull/bear
   cases but say they persist. Don't let overlays accumulate silently.

## Draft / scheduled entries

"Draft entries" here means the **forward-dated, unposted-to-reality** AR and AP
journals for invoices and bills you *expect* but haven't billed yet — the
committed cash flow contracts already imply. Economico has no separate
draft-journal status today; the **scenario overlay is the draft mechanism**:
entries posted inside it are excluded from real balances by construction, and a
future `external_timestamp` places them on the timeline. When the real
invoice/bill is eventually created on the live ledger, the scenario draft is
simply discarded (delete/reset) — it never double-counts because it was never
real. Derive one draft per obligation per period; keep it idempotent by tagging
each with a stable reference (e.g. `forecast:<obligation>:<period>`). See the
playbook for the table layout that mirrors the month-by-month projection reports.

## What to flag

- **Runway cliffs.** Call out the month cash crosses zero, with the assumptions
  that drive it (collection timing, churn, the modeled change).
- **Timing, not just totals.** A forecast that nets out fine annually can still
  have a cash trough mid-year — show the path, not only the endpoint.
- **Assumption sensitivity.** State the few inputs the answer hinges on (price
  delta, churn rate, collection lag) and, when it matters, branch a `bull`/`bear`
  clone rather than asserting one number.
- **Committed vs speculative.** Separate entries derived from real obligations
  (committed) from the what-if you layered on (speculative) so the reader knows
  which is which.

## Output

A planner's writeup, not a data dump:

- The headline answer first, in the business's currency.
- The projected statements (base vs scenario), and the runway / MRR / margin
  deltas.
- The assumptions, called out explicitly.
- The scenario name and the tool calls (or CLI commands) you ran, so a human can
  reproduce or re-run it.
- A clear statement that **the real ledger was not touched** — all of this lived
  in a scenario — and whether you deleted it or left it for reuse.

For the forward-entry derivation, the runway/cash-flow formulas, and the
draft-entry table format, read [`references/playbook.md`](references/playbook.md).
