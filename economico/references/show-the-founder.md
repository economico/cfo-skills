# Show the founder their business

The first session is the moment the founder sees their company as a set of books for the first
time. Show it as it takes shape, one view at the end of each phase, rather than one summary at
the end. Every view comes from a read of the ledger or from the brief, never from memory.

## Two ways to show a view

- **Inline, in hosts that render MCP Apps** (Claude on the web and desktop, and other hosts that
  support MCP Apps). The results of `reports`, `subjects` and `parties` carry Economico's app:
  the host draws the statement, contract, price book or list under the tool call. Make the read, then add one line
  saying what to look at ("revenue flows left to right into cost of revenue and margin").
- **As text, everywhere else** (Claude Code, Codex and other terminal agents, or when you cannot
  tell). Write the text version of the same read into your message: a short Markdown table built
  from the tool result, amounts scaled once by the ledger's `decimals`. Do not draw ASCII charts
  or send the founder to a web page instead; the text version is the view.

If you cannot tell which kind of host you are in, show the text version. A picture the founder
sees twice costs a few lines; a picture they never see costs the moment.

The views are part of what the session produces, not a courtesy. Show them even when the founder
told you to proceed without review, and even when no one is watching: a founder who runs you in
the background reads the session afterwards, and the views are what they read. A progress note
("recording now", "the books balance") is not a view.

## What to show, phase by phase

| Phase | View | Read | Text version |
|---|---|---|---|
| Propose | The model before anything is recorded (in the review message; when told to proceed, at the top of the final message) | `reports run model_preview` with the brief's tables as `model` | The price book and the cost table from `business-model.md`, plus the run-rate line (money in, money out, per month) from the preview's `run_rate` |
| Record | The price book as recorded | `subjects {action: "list", type: "template", plan: "on_sale"}` | One row per plan: plan, charge, price, when it bills |
| Record | Who the business trades with | `parties {action: "list"}` | One row per party: name, customer or vendor, contact |
| Record | The contracts as they land | `subjects {action: "list", type: "contract"}` | One row per contract: party, customer or vendor, template, status, since |
| Record | One contract that needed a judgment | `subjects {action: "get", type: "contract", id}` | The contract's terms, its activities and what each one posts, in a few lines |
| Prove | How the business makes and spends money | `reports run income_statement` with `from` and `until` | The flow in one line, then the table (below) |
| Prove | Where the money is | `reports run balance_sheet` with `as_of` | Assets, liabilities and equity with their main lines, and the check that they balance |
| Prove | Who owns the company | `reports run cap_table`, and `capital_accounts` for an LLC, partnership or project | One row per holder: name, instrument, units, ownership percentage, vested; then each owner's capital balance |
| Prove | Who owes whom | `reports run aging` with `direction` `receivable`, then `payable` | One row per open invoice or bill: counterparty, amount, due date, bucket |
| Prove | Recurring revenue | `reports run saas_metrics` with `year` and `month` | MRR, ARR and ACV, then gross margin, burn and runway, each with its basis; an undefined metric says why |
| Prove | What bills next | `reports run activity_plan` with `contract_id`, `from` and `through`, for the contracts the founder asks about | One row per upcoming activity: due date, activity, period, what it posts or what it waits for |

Show the propose view in the review message; it is what the founder reviews. When the founder
told you to proceed without review, show it at the top of the final message instead. Show the record
views once, after contracts are recorded, not after every command, and repeat the contracts table
at the top of the final message with the prove views. In the app, the price book reads each
charge's price from the charge itself, so it shows the prices the templates were built on. Show the prove views in the
order above; they are the "What the books say now" section of the brief, as pictures.

## The text versions

**Income statement.** Lead with the flow, the text counterpart of the app's flow diagram, then the
lines that make it up:

```text
Revenue $6,000 → cost of revenue $1,800 → gross margin $4,200 → operating expenses $2,400 → net income $1,800
```

| Revenue | | Expenses | |
|---|---|---|---|
| 4110 Subscription revenue | $4,800 | 5220 AI and model inference (cost of revenue) | $1,500 |
| 4120 Usage-based revenue | $1,200 | 5210 Hosting and cloud infrastructure (cost of revenue) | $300 |
| | | 5320 AI and developer tools (research and development) | $400 |
| | | 5620 Legal and corporate fees (general and administrative) | $2,000 |

(An invented business: the figures only show the shape. Yours come from the report.)

**Balance sheet.** Totals first, with the check stated, then the lines a founder recognizes:
cash per account, receivables, deferred revenue, payables, what is owed to the founder.

**Cap table.** A table, never a sentence: one row per holder with the instrument, units,
ownership percentage and vested units, then a row per SAFE or note with its holder, amount and
"converts at the next priced round" (no share count, no percentage). For an LLC or a project, add
each owner's capital balance from `capital_accounts`.

| Holder | Instrument | Units | Ownership | Vested |
|---|---|---|---|---|
| Ada Park | restricted common | 6,000,000 | 66.7% | 0 |
| Ben Ito | restricted common | 3,000,000 | 33.3% | 0 |
| Hartwell Ventures | SAFE, $250,000 | — | — | converts at the next priced round |

**Aging.** Only open items. A receipt paid at purchase or on a card never appears; say so if the
founder expects to see one.

**Contracts.** Keep the Economico service agreement off the list or mark it as Economico's own;
it is not part of the model you built.

**The proposal.** In hosts that render the app, the preview draws the price book, the
customers, the costs with their accounts, what drives each cost and the run rate under the call;
add one line saying what to review ("check the Globex price and the vendors marked cost of
revenue"). Everywhere else, the text version is the brief's tables with the preview's figures.

## What there is no view for yet

Say it in one line rather than improvising a picture: unit economics by product, plan and
customer is not available yet. Use the text version for it.
