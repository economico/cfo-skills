# Changelog

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
