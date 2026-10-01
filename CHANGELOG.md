# Changelog

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
