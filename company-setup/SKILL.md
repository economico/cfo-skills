---
name: company-setup
description: >
  Set up and configure a business on Economico — the guided first-run flow once
  the ledger is connected, and the home for each setup task it covers: the
  legal-entity profile and one-line description, named bank/wallet accounts, and
  the cap table (share classes, founder share issuance, early SAFEs). Use this
  skill rather than hand-running the tools whenever someone wants to set up or
  configure any of these — a full onboarding or just one piece; it sequences the
  tools and confirms before anything posts. Triggers: "set up my company /
  business", "onboard my business", "first-run setup", "set up my cap table",
  "issue founder shares", "record a SAFE", "add my bank account / Mercury / my
  wallet", "set our legal entity type / company profile". If the ledger isn't
  connected yet, use setup-economico first. Sets the company up, then hands off —
  pricing to pricing, contracts to creating-contracts, vendor receipts to
  expense-tracking. Not for connecting, billing customers, or reading the books.
---

# Company Setup (guided first-run)

Connecting Economico only *authenticates* the ledger. This skill is the next step:
it configures the **business** — the profile, the cash accounts, and the cap
table — so a founder comes out of it with a correctly set-up company without
having to know which tool to call in which order.

Work top-to-bottom, but skip anything that's already done. **Read before you write;
confirm before anything posts to the books.** Everything here can run against the
seeded `test` scenario first (pass `scenario: "test"`) to rehearse — see
`setup-economico`.

## 0. Preconditions — connected and un-gated

- **Connected?** This skill assumes an authenticated session. If `get_business`
  401s or no tools are visible, stop and use **`setup-economico`** first.
- **YC gate.** A brand-new account is gated to **read-only** until it's verified as
  a YC founder — every write step below (`update_business`,
  `create_financial_account`, `create_share_class`, `issue_shares`, `record_safe`)
  is blocked until then. Call `get_business`; if it isn't YC-verified, walk the
  founder through `verify_yc(link)` first (the link is minted at
  `https://bookface.ycombinator.com/verify`). The read tools this skill uses to
  inspect state (`get_business`, `list_parties`, `list_financial_accounts`,
  `list_share_classes`, `get_cap_table`) work before verification, so you can
  survey what's already set up either way.

## 1. Company profile

`get_business` to see what's set, then fill the gaps with `update_business` — a
partial update, only the fields you pass change:

- `name` — legal/display name.
- `legal_entity_type` — one of `llc`, `c_corp`, `s_corp`, `sole_proprietorship`,
  `partnership`, `nonprofit`, `unincorporated`, `other`.
- `jurisdiction` — where it's incorporated, e.g. `US-DE`.
- `description` — a YC-style one-liner of what the company does (≤ 280 chars).
- `url` — the company website (`http(s)`).

This flows onto invoices and the hosted business pages, so it's worth getting right.

## 2. Bank & wallet accounts

Register each real cash account the business holds so the agent can pick the right
one for payments. Each is a named account mapped to a GL cash code, with its own
sub-balance.

For each account:

1. **Resolve-or-create the provider party** (the bank / processor / wallet host):
   `list_parties` → reuse a match, else `create_party(name, url)` (e.g. "Mercury",
   `https://mercury.com`). Linking it here is what classifies it as a
   financial-services provider.
2. `create_financial_account(name, provider_party_id, gl_account_code, currency,
   kind?, last4?, network?)`:
   - `name` — how the founder refers to it, e.g. "Mercury checking".
   - `gl_account_code` — `1110` for a fiat bank account, `1310` for a crypto wallet.
   - `currency` — ISO 4217 (`USD`) or a CAIP-19 asset for wallets.
   - `kind` — `checking` / `savings` / `wallet` / `other`; `last4`, `network`
     (`ach`, `base`, …) optional.

Two accounts in the same currency (Mercury **checking** + **savings**) each keep a
distinct balance — register both. `list_financial_accounts` to confirm.

**One pool, many identifiers.** An account is one balance; the ACH details, IBAN,
or blockchain addresses that reach it — and their per-rail fees and speeds — are a
separate layer. A modern custodial account (Stripe's financial account) holds one
balance reachable via ACH *and* several chains; register it as **one**
`create_financial_account`, then add its identifiers and priced in/out methods
(plus account-level card/Flow collection). Don't register a second account per
rail — that splits one real pool into two GL balances and breaks reconciliation.
Registering those rails (and a counterparty's payment instructions, and routing)
is its own step — hand off to **`payment-rails`**.

## 3. Cap table

A share class is the authorized ceiling; issuances draw against it and post to the
GL. Amounts are **minor units** (cents).

1. `create_share_class(name, authorized_shares)` — e.g. `"Common"`, `10000000`.
   Add `"Preferred Seed"` etc. later as needed.
2. **Founder issuance** — for each founder, resolve-or-create their party, then
   `issue_shares(party_id, holder_role="founder", share_class_id, quantity,
   consideration_amount, currency, occurred_at?)`. `consideration_amount` must be
   **positive**; for founder grants use a nominal par value (e.g. 5,000,000 shares
   at $0.0001 par = $500 = `50000` cents). Posts **Dr cash / Cr 3100 Owners'
   Equity**. A 50/50
   split is just two equal issuances.
3. **Early investors / SAFEs** — `record_safe(party_id, amount, currency,
   valuation_cap_minor?, discount_bps?)`. A SAFE is a financing liability, **not**
   shares yet — posts **Dr cash / Cr 2240 SAFE Financing**; no dilution until it
   converts. `discount_bps` is basis points (`2000` = 20%).
4. `get_cap_table` — confirm ownership basis points and authorized-vs-issued per
   class read the way the founder expects.

See [`references/cap-table.md`](references/cap-table.md) for the par-value math,
share-class choices, and SAFE terms.

## 4. Hand off — the rest of the setup lives in sibling skills

Once the profile, accounts, and cap table are in place, this skill's job is done.
Route the founder to the skill that owns each next step — don't re-implement them
here:

- **Payment rails** — the identifiers that reach each account (ACH/IBAN/wallet),
  their priced in/out methods, card/Flow collection, counterparty payment
  instructions, and cheapest/fastest routing → **`payment-rails`**.
- **Pricing** — turning the business model into a `pricing.md` and revenue
  obligations → **`pricing`**.
- **Customers** — order forms and customer contracts before billing →
  **`creating-contracts`** (then **`invoicing`** to send bills).
- **Vendors & receipts** — "scan my email for bills / receipts", building the
  vendor spine, recording bills → **`expense-tracking`**.
- **Reading the books** — reports, runway, investor metrics →
  **`financial-analyst`** / **`investor-reporting`**.

Use [`references/checklist.md`](references/checklist.md) as the ordered first-run
checklist and a map of which skill owns each later step.

## Discipline

- **Read before write.** `get_business`, `list_parties`, `list_financial_accounts`,
  `list_share_classes`, `get_cap_table` before creating anything — resolve-or-create,
  never blind-create.
- **Confirm before the books move.** `update_business`, `create_party`, and
  `create_financial_account` are safe metadata. `issue_shares` and `record_safe`
  **post GL journals** — confirm the shares, amounts, and par with the founder
  first, and prefer rehearsing in the `test` scenario.
- **Units.** Money is minor units (cents): `$0.0001 → nothing`, `$1.00 → 100`.
  Share quantities are whole shares, not micros.

## References

- [`references/cap-table.md`](references/cap-table.md) — authorized vs issued,
  founder par-value math, common vs preferred, SAFE cap/discount, GL effects.
- [`references/checklist.md`](references/checklist.md) — the ordered first-run
  checklist and which sibling skill owns each later step.
