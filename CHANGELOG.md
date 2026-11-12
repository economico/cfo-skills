# Changelog

## 1.13.0 (2026-11-12)

- Custom reports: an answer the founder will want again is saved as a report Economico draws from a definition the agent writes, with charts, tables, figures and text, and parameters such as the month. The verify reference teaches the order: `report_format {action: "guide"}`, `report_sql {action: "schema"}`, `report_format {action: "check"}`, `reports {action: "preview"}`, then `report_definitions.publish`.
- Opening a saved report again: `reports {action: "run_definition", id, params}`, reading a parameter's `note` when a value fell back to the default, and editing a report by reading its source back with `report_format {action: "source"}` and publishing it under the same `id`.
- How current a run is: an answer may be reused for up to an hour while the books are unchanged; `computed_at` says when it was worked out, and `fresh: true` forces a new run.
- Pins and links: `report_definitions.pin` puts a report in every member's workspace sidebar, and with `overview: true` also on the overview; a pin follows the latest version. `shared_views.create` sends a report as a `frozen` link that keeps the figures for the month chosen, or a `live` link that follows the books and lets its reader change the parameters.
- The investor update is built as a custom report with a `month` parameter, published as a new version each month and sent as a frozen link; the Markdown update is the fallback where the connection may not publish reports or sharing is not on the plan. `/investor-update` describes it that way.
- Vendor plans from the catalog: before modelling a vendor by hand, the agent looks it up with `vendor_catalog` (`find`, `plans`, `history`, `get`) and adopts the plan version that was in force when the contract started, using `contracts.adopt`. It checks the selected seller against the invoice, chooses each right's expense account, and passes `replaces` to supersede contracts that were recorded by hand. The vendor-contracts reference sets this order under "Look up and adopt first", and `/record-vendor-invoice` adopts a vendor plan before building a contract.
- Vendor terms at adoption: different commitment, renewal, notice or cancellation terms go in a complete, evidenced `lifecycle` on `contracts.adopt`, never a derived template. The agent reads the adoption's `lifecycle_terms_differ` and `lifecycle_terms_unknown` flags and keeps unknown terms as an open gap.
- Plan changes stay inside one contract: the agent reads every invoice before recording any of them and asks the founder about each upgrade, downgrade or credit for unused time. It then amends from the change date with `contracts.amend` and its `proration` lines before recording anything dated later, including the recognition of the old period. If recognition was recorded too early, `contracts.reverse_economics` with `recognitionOnly: true` undoes it. Any other blocking record is reported as an open gap. No phase is dated after today, and the vendor subscription and annual-prepay recipes say to amend first.
- Seat changes: for a right that declares `economics.quantity`, `contracts.change_quantity` records the new count. Its proration and decrease rules set the charge or credit, and an optional evidenced bill records the charge in the same transaction. For GitHub Team, the bill takes the before-tax amount, states tax separately and is recognised by `changeId`. Customer MRR reads the dated count in force.
- Reissued vendor bills: `replacesBillId` on `contracts.record` or `contracts.reconcile_usage` replaces a bill the vendor reissued against a replacement plan, all in one transaction. Payments already made become vendor credit applied to the new bill. The payments reference explains how `payments.unallocate` and `payments.reverse` work once a bill has been replaced.
- People and provider payroll: a new reference covers PEOs, EORs, payroll processors and direct contractors, with Deel, Remote, Rippling, Justworks and Gusto as starting cases. The agent records team membership and arrangements (`people.set_membership`, `people.record_arrangement`), provider invoices and payroll registers (`payroll.record_statement`), later detail on costs and links to costs already in the books (`payroll.attribute_cost`, `payroll.link_recorded_cost`), prefunding and deposits (`payroll.link_funding`), and provider credits (`payroll.reverse_statement`). It reads them back with `people_costs` and `payroll_provider`.
- Subscription review: a new vendor usage reference has the agent record seat and tool use on a free meter (`contracts.add_usage_meter`, then `contracts.observe_usage` once per billing cycle). It lists where usage data lives for GitHub, OpenAI, Anthropic, Cursor, Vercel, AWS, Google Workspace, Slack and Notion. The review starts from the `vendor_utilization` report and reports unused, over-provisioned, about-to-change and unknown subscriptions, each with its saving. Named seats link to people with `people.link_identity`.
- Detailed vendor usage: past usage is backfilled from the vendor's own reports with `contracts.observe_usage` on the adopted meter, and quantities are converted explicitly to the meter's unit. A unit that differs from the meter's is flagged. Invoices are reconciled with `contracts.reconcile_usage`. For GitHub Actions, the agent records each charge once, either as an amount or as metered minutes, and the overlap between the two is flagged as `invoice_overlap`.
- Startup credits: a grant such as a YC cloud credit is recorded as a granted credit lot, first `contracts.add_granted_credit` and then an `exercise` on the new right. The agent reconciles it with the vendor's statements and tells the founder the balance left, when it expires and what each service costs without it. Adoption refuses to replace a contract that still holds credit unless `retireCredit: true` is passed.
- Stripe fees: `payments.process` records a gross receipt together with its processing fee. Billing, Invoicing, Tax and Radar fees taken from the balance are billed once per balance transaction (`txn_…`), and the agent matches existing bills and old ID forms first. Payouts are recorded with `payments.transfer`, and overlapping fees are settled through `contracts.resolve_usage_review`.
- Company cards and issuer statements: a card is registered on 2190 with its issuer contract, and the statement closing day, payment due day and autopay go in the contract's `paymentTerms`. The agent records the issuer's statement with `card_statements.record`, repayments with `card_statements.pay`, and corrections with `card_statements.void` and `card_statements.reverse_payment`. Purchases can also be booked as a monthly `journal.post`. For any vendor contract, `paymentTerms.payingAccountId` sets the account bills are paid from by default.
- Opening balances: a bank account, card or wallet that already holds money starts with `openingBalance` and `openingDate`, offset to `3900` Opening balance equity. An account registered earlier and never used takes one through `accounts.set_opening_balance`.
- Watching a Tempo wallet: `accounts.monitor_wallet` registers a company wallet's token by address only, with its opening balance read from the chain. Economico reads it daily, or now with `accounts.check_wallet_monitor`; `accounts.classify_wallet_movement` records an unknown movement once the founder says who it is from and what it pays, while a movement to another company account is a `payments.transfer` and cash already recorded by hand is never classified again; `accounts.set_wallet_monitoring` pauses or resumes it, and `accounts.reconcile_wallet_monitor` resumes after a chain reorganisation. The monitor's state and movements are read through `accounts`.
- More report kinds in verify: `owner_balance`, `income_statement_periods`, `unit_economics`, `contract_books`, `pipeline`, `concessions` and `platform_fees` each answer a question in the checks table, and the wallet monitor's reads prove a watched wallet agrees with the books.
- Stablecoins and token wallets: stablecoin accounts go on `1410`, whether the business or a provider holds them, and the report decides how they are classified. A wallet the business holds itself books each token in its own ledger, at the precision its contract reports, with the token's CAIP-19 code as `currency`. Tokens are moved or sold with `payments.transfer`, and holdings are read from `accounts overview`.
- A founder's card in another currency: the card is registered once, in the company's currency. A bill in another currency is paid in that currency and recorded at the bill's booking rate, which is what the founder is owed, never the card statement's amount. A vendor's refund reduces what is owed at the same rate. Two new recipes show this: founder card, foreign bill and founder card, foreign refund.
- Answering a one-off question from the books: when no report answers it, the agent runs one read-only `report_sql {action: "query"}` after `report_sql {action: "schema"}`, instead of adding up several reads itself. It follows the money, currency, ledger and date rules in the verify reference, and checks its cut against the report's total. The aging check in verify counts 2170–2172 payables beside 2110.
- Publishing a plan to the vendor catalog, only when the founder asks: `plan_publications` with `preview`, then `publish` once the founder agrees, a new version with `supersedes`, and `withdraw` to stop new adoptions. Business model patterns come from the catalog's `patterns` shelf. A brief's `catalogSource` points to the catalog version whose `activity` is the shape to use.
- Logos: a customer's or vendor's logo is uploaded with the `logos` tool (`upload`, `list`, `replace`, `delete`), only from an image the founder supplied or approved, and set as the party's `logoUrl`. The business's own `logoUrl` puts its logo on its share pages.
- AgentID sign-in: an agent with an AgentMail inbox can choose **Continue with AgentID** in the OAuth sign-in. The connect reference links to the procedure.
- Plan choices: a Solo refusal offers Founders, while the team fits it, or Startup through `details.choices`, and shared views and extra members need a paid plan. The import allowance is 50,000 events on Startup while that plan is active.
- The founder's business date sets the latest date the agent records anything for.

## 1.12.1 (2026-11-10)

- Hermes Agent plugin catalog prep: `plugins/mcp/` ships an Agent Plugins v1 package (`plugin.json`, `mcp.json` → `https://economi.co/mcp`, README, `economico-books` skill) so Economico can be listed in NousResearch/hermes-agent after `skills:publish`.

## 1.12.0 (2026-11-02)

- Voiding a contract set up in error: a new "void a contract set up in error" section in the vendor-contracts reference walks the preview, approval, `contracts.void`, rebuild and `payments.allocate` steps, and the contract and how-it-works references point to it instead of correcting phase by phase.
- Party roles come from the agreement: the product-modeling reference tells the agent to declare a party's relationship (customer, vendor, investor, founder, employee) in the template's `partyCategories` and to read the party back after acceptance.
- Founders and Startup plans: the connect reference describes the Founders offer (three people in total, scoped agents included) beside Startup's unlimited people, and the import allowance (10,000 events) applies to Founders as well as Startup.
- The SAFE recipe checks the cap table's terms at the cap: a post-money cap, a discount and the ownership estimate at the cap, kept apart from fully diluted ownership.

## 1.11.2 (2026-10-23)

- Cursor can install the plugin: `.cursor-plugin/plugin.json` declares the same skills, the Economico MCP server and the icon, and leads with the same "ERP for ambitious founders and their agents" description and keywords as the Claude listing.

## 1.11.1 (2026-10-23)

- Reports for a month, quarter or year end with the next period's first instant as a UTC timestamp (`2026-10-01T00:00:00Z` for September), in the investor update and every recipe. A timestamp is read the same way before and after Economico's report dates came to include their whole day, so September never picks up 1 October's entries.
- The Claude plugin listing and README lead with "Economico: The ERP for ambitious founders and their agents", add `erp` and `agents` keywords, and describe Economico as the founder's own source of truth beside their bank and bookkeeper. The Codex and ChatGPT listing copy is unchanged while the ChatGPT submission is in review.

## 1.11.0 (2026-10-22)

- Explore the books before connecting your own: plugin listings show real TinyCRM demo views for recurring revenue, overdue invoices and ownership, with prompts to try.
- The ChatGPT upload includes five tested positive cases and three negative cases. Reviewer credentials stay in the private submission portal.
- Claude plugin metadata includes documentation, support, privacy, terms and an icon; the connector submission kit supplies the screenshot carousel and paired prompts.
- The ChatGPT skill explains priced service rights, contract acceptance, separate billing and recognition, and component-qualified payment allocations, with a fixed-price authoring example.
- The shared Claude and Codex skill preserves bank-deposit source identities, uses permitted workspace file tools for evidence, separates completed-service recognition from billing cadence, and saves evidenced plans before the first customer, keeping future-launch offers off sale.
- Company setup follows published issuer links for account branding. Founder review shows the model before asking material judgments, keeps missing ownership facts as gaps, and completes discovery, recording and verification in the foreground with a self-contained report.
- Keep unidentified card issuers unknown, distinguish financial providers from vendors, and repeat complete statement tables in the final response.
- Settle only unanswered scope choices before discovery, ask for model review after the brief and preview, and use the founder's stated business date for recording and reports.
- Receipt reviews discover zero-charge notices and explain why they were skipped; missing ownership and registration evidence is requested in the final message instead of offered as factual multiple-choice questions.
- Final reports list material decisions directly and format monetary amounts from ledger decimals; completed-service recognition remains delivery even when billing is monthly.
- Explain native books and returned indicative totals in investor updates, founder views and account setup, including missing rates and historical provenance.
- Add an optional market-rate check for independently evidenced payment amounts; it never supplies an amount or posts an invented difference.

## 1.10.3 (2026-10-21)

- The ChatGPT plugin package can be used without an existing business: someone without one creates it on the free Solo plan while connecting. Its skill and README say so. The Codex and Claude skills are unchanged.

## 1.10.2 (2026-10-20)

- Before a batch, `business {action: "meter"}` shows the same ceiling as the period's `allowed` while the monthly cap holds; the connect reference says the billing page tells the founder whether a cancelled plan can be kept.
- Verifying history: while derived contract values are still catching up, `events verify` returns `ok: false` with empty `diffs` and a `pending` list. The core state matched; only a non-empty `diffs` is a real difference.

## 1.10.1 (2026-10-19)

- Bringing in a contract from before the business joined: Economico's daily schedule never invoices its past periods. It records only what falls due after the contract is recorded, however long after signup that is, so the agent imports every earlier invoice, service period and expiry from the old books without duplicates.
- Economico's own plan invoices arrive in the books by themselves: the import, connection and vendor references say never to record or import them, and recording one is refused (`validation`, `details.reason: "economico_delivers_plan_invoices"`).

## 1.10.0 (2026-10-18)

- Bring in existing books free: a new import reference explains that history recorded at its own dates, before the business joined Economico, is a historical import. It does not count toward the monthly events and uses the one-time import allowance; the agent reads `business {action: "imports"}` for the cutoff and what is left, and each receipt says `imported: true`. Older packs work too: their history is imported the same way when it is dated before the business joined.

## 1.9.4 (2026-10-17)

- Add OpenAI plugin metadata and MCP configuration alongside the Claude plugin. The skill folders and existing installation paths are unchanged.

## 1.9.3 (2026-10-17)

- Startup is on sale: troubleshooting no longer calls the Startup features behind `plan_required` unavailable.

## 1.9.2 (2026-10-14)

- The plans have new names: Free is now Solo, and Team is now Startup. The references use the new names; prices, features and the `plan_required` refusal are unchanged.
- Troubleshooting explains the `plan_required` refusal for a used-up monthly Solo allowance: tell the founder the limit and the reset date, then retry after it or once they upgrade, rather than treating it as a permanent feature gate.
- Stripe package pricing (`transform_quantity` with `divide_by`) maps to one tier with `unitsPerPrice`: rounding up is `startedUnits` (every started package charged in full), rounding down is `wholeUnits`.

## 1.9.1 (2026-10-10)

- The listing describes the current pricing: free for a single founder, with no fee on revenue.

## 1.9.0 (2026-10-10)

- Paying a bill or collecting an invoice in another currency than the business's own realizes the foreign-exchange gain or loss in the same step: record only the evidenced amounts, and the books derive the difference.
- A vendor that bills in another currency is booked at the rate of the day the bill is first recorded. When no rate is on file for that day, the skill asks for the rate your bank statement shows instead of guessing one.
- The Stripe gross/net payout recipe no longer lists an empty account in its expected balances.

## 1.8.0 (2026-10-07)

- The "How Economico works" reference explains connecting with another business (`connections.request` and `connections.accept`) and waiting for a message's record to say `delivered`.
- The `economico` skill's "How Economico works" reference now tells the agent to report what is broken, unclear or missing to
  the Economico team with `messages.send`, as a support request or feature suggestion.

## 1.7.0 (2026-10-06)

- Settle claims and exchange cash across currencies with both evidenced amounts, each in its own currency's books; cancellation refunds follow the active books.

- Model founders’ restricted shares as one ownership right with independent funding, exact par/premium allocation and scheduled vesting; all original executable recipes now use priced rights.

- Model project ownership and repayable advances without invented shares or convertible instruments.

- Model LLC subscriptions, funding, owner draws and year-end allocations independently from payments.

- Derive priced overage from evidenced included-call consumption; invoice and collect separately.

- Model prepaid credits as one priced right: separate billing/payment, native lots and exact value release on usage.

- Replaced platform-plus-usage and Stripe recipes with priced services, coordinated processor fees and independent net payouts.

- Model services as priced rights with separate billing, recognition and payments.
- Added payment, reimbursement and correction guidance; vendor recipes now use independent
  bank/card/owner funding without a founder-expenses contract.
- Added an evidenced owner cash contribution recipe without invented shares; preserve supplied
  source identities and keep transaction facts separate from opening carry-forward journals.
- Settle existing historical claims through independent payments without recreating invoices.
- Added a convertible financing commitment recipe with separate funding and no invented equity.
- Added approved capital distributions with separate payments and holder balance safeguards.
- Added owner/card advances with later bill matching and separate reimbursement.
- Added financing debt conversion and repayment, with component-qualified claims and cap-table reconciliation.
- Close evidenced accumulated owner draws as a period consequence without another payment.
- Allocate evidenced retained profit or loss to existing members and select matching period consequences explicitly.
- Trace owner/card refunds to their original funding and recover refunds received after reimbursement, with an executable example.
- Replaced annual customer and insurance recipes with priced rights, independent payments and equal monthly recognition.
- Amend prospective service terms with evidence; the seat-expansion recipe preserves earlier prices and collections.
- Replaced consulting and SAFE recipes with priced rights and independent payments; reconcile commercial receipts in revenue summaries.
- Added a priced monthly subscription recipe with historical MRR/ARR and separate collection.
- Clearly distinguish validated new workflows from historical recipes still being replaced.

## 1.6.0 (2026-10-06)

- Added a consulting recipe for monthly hourly billing and a fixed milestone. Both customer
  receipts settle the billed contract claims through receipt activities.

## 1.5.3 (2026-10-06)

- The plugin directory listing now opens "Discover the business behind your product".
  Nothing changes in the skills.

## 1.5.2 (2026-10-03)

- The plugin directory listing now says what Economico does for you, with search keywords and a
  link to Economico's privacy policy (`https://economi.co/privacy/`). Nothing changes in the
  skills.

## 1.5.1 (2026-10-03)

- The Claude Code plugin declares only Economico's own MCP server. It no longer adds Stripe:
  connect Stripe yourself if you want your agent to read it, as before 1.5.0.

## 1.5.0 (2026-10-02)

- Installing the Claude Code plugin now connects the MCP servers the skills work through:
  Economico (`https://economi.co/mcp`) and Stripe (`https://mcp.stripe.com`). Sign in to each
  with OAuth from `/mcp`. If you added Economico with `claude mcp add` before,
  you can remove that entry.

## 1.4.2 (2026-10-02)

- Connecting the ledger always grants full access to each business you approve, up to your
  own permissions; there is no access level to pick on the consent screen any more. The
  skill no longer tells you to choose Read books, Keep books or Administer, and asks the
  business's owner to connect when the legal-record commands are missing.

## 1.4.1 (2026-10-02)

- The Claude Code plugin now has its own manifest, `.claude-plugin/plugin.json`, listing its
  skills, and an icon. The marketplace entry only points at it. Installing works as before.

## 1.4.0 (2026-10-02)

- Backfilled history now lands in revenue in the same session. The skill records each
  contract's history as one timeline in date order (bills, payments at their own dates, usage,
  and each completed month's service) instead of leaving completed months to the ledger's
  daily timer, which kept them in deferred revenue until its next run. Verifying the books
  now checks that no period due before today is left unrecorded.
- A loss year can now be allocated to the owners' capital with the new `allocate_loss`
  pattern, the way a profit year already is with `allocate_profit`. The project-owner recipe
  allocates its year's loss again, so its `capital_accounts` shows nothing left
  undistributed.
- Contracts are created at their real start date (the signing, the subscription's start, the
  formation date) rather than today, so backfilled history is never refused for predating
  the contract; a draft created at the wrong date is discarded with `contracts.discard` and
  created again.
- Economico's address is now `https://economi.co`: connecting over MCP and logging in with the
  CLI use it.

## 1.3.0 (2026-10-01)

- Every first session now records who owns the company. The skill reads the legal form and
  looks for the formation documents (certificate, stock purchase agreements, operating
  agreement, a Stripe Atlas confirmation), sets the legal form if it is missing, and records
  each owner from the recipe for that form, so `cap_table` lists the holders even when there is
  a single owner who has put in no cash yet. When no document gives the quantities, it asks
  instead of leaving the cap table empty.
- New procedure for a single-member LLC: one operating agreement, membership units authorized
  and allocated to the member, and the member's capital from cash, contributed founder-paid
  costs or draws.
- The brief gains an Ownership table, and the final message shows the cap table beside the
  statements: a table with one row per holder (units, ownership, vested) and each SAFE as an
  amount that converts later, never as shares.
- Accounts now carry their issuer's look: the skill researches each bank's, card issuer's or
  token's brand color and logo and sets them with `brandColor` and `logoDocumentId` (a logo is
  received as an `image` document, PNG, JPEG or WebP). The workspace draws a card in its
  issuer's color, a bank account under its bank's band, a wallet in its token's color.
- Payment details: when your mail gives an account's receiving details, the skill registers
  them as a payto link (ACH or IBAN) or, for a wallet, a CAIP-10 address with its token, so you
  can copy them from the account's page. It never guesses a number and never records a card
  number.

## 1.2.0 (2026-09-30)

- Before anything is recorded, the proposed business model is previewed with the new
  `model_preview` report: it checks the price book, customers and costs, computes the monthly
  run rate, maps which product actions drive each vendor's cost, and draws the proposal inline in
  hosts that show Economico's app.
- `business-model.md` now ends with "For the next session": which business these books are,
  your standing answers, the rules you set and the questions still waiting for you. A later
  session reads it first, so it resumes instead of asking you again.
- The skill now also triggers when you ask it to pick up the books where an earlier session
  left off.
- New tested recipes: per-seat plans with a seat expansion, included usage with overage,
  prepaid credit packs, an annual vendor plan paid upfront and released monthly, and Stripe
  card payments at gross with fees and net payouts.

## 1.1.0 (2026-09-30)

- Add payment-account and provider-agreement procedures for registering payment methods and
  recording provider fees with completed payments. Clarify that the selected method's fee is
  already posted and must not be recorded a second time.
- Every party gets its role when it is created: founders, LLC members, partners and project
  owners as `founder`, SAFE and priced-round investors as `investor`, employees granted stock
  as `employee`. No contract sets a party's role, so a party created without one was listed
  under "Other". The SAFE, LLC members and project owner recipes now set theirs.
- `documents.receive` takes an optional `title`, the founder's word for the document; the
  evidence reference shows it and says when to leave it to the file's own heading or PDF
  title.

## 1.0.0 (2026-09-30)

A new skill set for Economico's current tool surface, where every write is a command from one
catalog.

- `economico`: one skill that connects Economico, models your business from your code, Stripe
  and email, records it, and proves the books with reports. It asks you its questions as
  choices with a recommendation, and replaces `setup-economico`, `company-setup`, `pricing`,
  `creating-contracts`, `invoicing` and `expense-tracking`.
- Slash commands for the requests you make most: `/add-customer`, `/send-invoice`,
  `/record-payment`, `/record-vendor-invoice`, `/record-receipts` and `/investor-update`.
- Removed: `payment-rails`, `financial-analyst`, `investor-reporting`, `cpa` and `forecasting`.
  They called tools the current Economico does not have. `v0.1.0` and the `go-era` branch keep
  them.
- One Claude Code plugin, `economico`, installs every skill.
