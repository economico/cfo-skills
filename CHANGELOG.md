# Changelog

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
