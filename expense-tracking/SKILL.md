---
name: expense-tracking
description: >
  Ramp-style accounts payable for Economico: find vendor receipts and invoices in
  the user's connected email and record them as bills — but first build each
  vendor's books-grade spine (a vendor party, an active vendor contract from its
  terms/pricing pages, and obligations carrying the chart-of-accounts code + SKU)
  so every line maps to the right account and feeds unit economics. Use when asked
  to process receipts, track or categorize expenses, run accounts payable, or
  "add this invoice / bill". Triggers on: "track expenses", "process my receipts",
  "log this invoice", "record this bill", "categorize our spend", "accounts
  payable", "what are we paying for", "scan my email for receipts", "add the AWS /
  GitHub / OpenAI invoice", "book this vendor bill", "set up our vendors". Not for
  analyzing the numbers (use financial-analyst), billing customers (use
  invoicing), or connecting Economico (use setup-economico). Assumes Economico is
  connected and the agent can search email.
---

# Expense Tracking (Ramp-style AP, with a books-grade vendor spine)

Most expense trackers just OCR a receipt and dump a number into a category. We do
something stronger: for every vendor that bills us we build a **party → contract →
obligation** spine in Economico, so each invoice line posts to the right
chart-of-accounts code (with a SKU sub-key where it matters). That turns
accounts payable into business intelligence — the same records that close the
books and reconcile payments also power margin and unit-economics analysis later
(see the `financial-analyst` skill).

Work the loop: **find → identify the vendor → build/confirm the spine → record the
bill → (optionally) pay**. Read before you write; confirm before anything that
moves the books.

## 1. Find the receipts (email)

Use the agent's existing email connection (e.g. a Gmail MCP server or the host's
mail tools — this skill is host-agnostic). Search the inbox for vendor receipts
and invoices: queries like `from:(billing OR invoice OR receipt OR no-reply)
subject:(invoice OR receipt OR payment OR "your bill")`, plus known vendor
domains. For each candidate, extract: vendor name + domain, invoice/receipt
number, issue + due dates, currency, line items (description, qty, unit price),
and the total. Keep the message id so you can cite the source and avoid
re-processing it.

If the user handed you a single invoice (PDF/text/email) instead, skip the search
and process that one.

## 2. Identify the vendor → ensure a vendor **party**

- `list_parties` and match on name or the email domain (e.g. `aws.amazon.com`).
- No match → `create_party(name, url)` with the vendor's website so its DID
  derives (`did:web:<host>`). One party per real-world vendor; don't duplicate.

## 3. Build/confirm the vendor **contract** (the spine's backbone)

A vendor contract groups the vendor's obligations and records the terms you're
agreeing to.

- `list_contracts(party_id)` — reuse an existing **active** vendor contract if one
  exists.
- Otherwise `create_contract(party_id, role="vendor", currency, msa_url?,
  order_form_url?, services_scope?, term_length?, payment_terms?)`:
  - `msa_url` — the vendor's **terms-of-service URL** (vendor contracts accept the
    vendor's own terms; it's optional — omit if you don't have one).
  - `order_form_url` — the vendor's **pricing page** the charges come from.
  - `services_scope` / `term_length` / `payment_terms` — what you researched.
  - Then `update_contract_status(id, "active")` — obligations can only be billed
    under an **active** contract.

Research fills these in: the signup/welcome email, the ToS URL in the receipt
footer, and the vendor's pricing page tell you the plan, cadence, and what each
line is — see [`references/research.md`](references/research.md).

## 4. Map the spine: **obligations** carry the account code + SKU

Each recurring product/plan the vendor charges for becomes an obligation under the
contract, and the obligation is where the **GL account code** lives:

`create_obligation(contract_id, type, name, account_code, sku?, amount_minor_units?,
interval?, metric_name?, price_per_unit_minor?, source_obligation_id?)`

- `account_code` — the expense/COGS code for this spend (vendor obligations must
  use a 5xxx/6xxx code). Pick it from the receipt + pricing page; default
  sensibly when unsure. `list_chart_of_accounts(currency)` is the authoritative
  live list of codes; the categorized map is in
  [`references/account-codes.md`](references/account-codes.md).
- `sku` — set it on SKU-keyed COGS accounts (5300/5400/5600/5900) so spend tracks
  per product for unit economics (e.g. `sku="ec2"` on 5300).
- `type` — `recurring` (with `interval`) for subscriptions, `usage` (with
  `metric_name` + `price_per_unit_minor`) for metered spend, `one_off` for a
  single charge.
- `source_obligation_id` — when this cost directly serves a customer revenue
  obligation, link it so margin/unit-economics analysis can tie cost to revenue.

Reuse existing obligations on the contract for repeat charges; only create new
ones for genuinely new line types.

## 5. Record the **bill**

`receive_bill(party_id, contract_id, amount, currency, due_date, memo?, lines[])`
where each line is `{description, quantity_micros, unit_price_minor,
obligation_id?}`.

- Amounts are **minor units** (cents): `$49.99 → 4999`. Quantities are **micros**:
  `1.0 → 1_000_000` (10 hours → `10_000_000`). `amount` must equal the sum of the
  lines.
- Tie each line to its obligation via `obligation_id` so it posts to that
  obligation's account code + SKU. A line with no obligation falls back to the
  default COGS account (5900) — fine for a one-off, but prefer obligations for
  anything recurring.
- Put the source invoice/receipt number in `memo` for the audit trail.

Then **confirm with the user** and `approve_bill(id)` — that posts the journal
(Dr expense / Cr accounts payable). Before approving, `get_bills(party_id)` to
make sure you're not double-recording a receipt you already booked (see
[`references/dedup.md`](references/dedup.md)).

## 6. Pay (only when asked)

When a payment actually clears, `pay_bill(id, amount, currency, idempotency_key)`
posts Dr accounts payable / Cr cash. This **moves money** — confirm first, and
note the payment refuses to overdraw the source cash account.

## Discipline

- **Read before write**: `list_parties` / `list_contracts` / `list_obligations` /
  `get_bills` before creating anything — resolve-or-create, never blind-create.
- **Confirm before the books move**: creating parties/contracts/obligations and
  `receive_bill` (draft) are safe; `approve_bill` posts the GL and `pay_bill`
  moves cash — confirm those with the user.
- **Idempotent**: one bill per (vendor, invoice number / period). Rely on
  `receive_bill`'s external-ref idempotency and your `get_bills` check.
- **Cite your sources**: reference the email/receipt and the pricing/ToS pages you
  used so the books are auditable.

## References

- [`references/account-codes.md`](references/account-codes.md) — which expense/COGS
  code (and when to set a SKU) per kind of vendor spend.
- [`references/research.md`](references/research.md) — using signup emails, ToS,
  and pricing pages to fill the contract and choose codes.
- [`references/dedup.md`](references/dedup.md) — avoiding double-recording and
  handling repeat/recurring charges.
