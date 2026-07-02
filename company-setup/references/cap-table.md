# Cap table reference

How Economico's cap-table tools model equity, and the judgment calls to get right
during first-run setup.

## Share classes: authorized vs issued

`create_share_class(name, authorized_shares)` sets the **authorized ceiling** — the
maximum number of shares that class can ever issue. Issuances draw against it;
`issue_shares` is rejected if it would push issued past authorized. Authorized is
not ownership — a company can authorize 10,000,000 common and issue only the
founders' slice, leaving the rest for the option pool and future rounds.

- **Common** — founders and employees. Start here.
- **Preferred** (e.g. "Preferred Seed", "Series A") — priced/negotiated rounds.
  Add these when a round actually happens, not during first-run setup.

A typical early setup: one **Common** class, authorized ~10,000,000, with the
founders issued against it and the remainder left unissued for the pool.

## Founder issuance & par value

`issue_shares(party_id, holder_role="founder", share_class_id, quantity,
consideration_amount, currency, occurred_at?)`.

- `quantity` is **whole shares**.
- `consideration_amount` is the cash paid, in **minor units (cents)**, and **must be
  positive**. Founders don't pay "fair value" for founder stock — they pay a nominal
  **par value**. Par is per-share and tiny, e.g. $0.0001.
  - 5,000,000 shares × $0.0001 = $500.00 → `consideration_amount = 50000` cents.
  - If you truly want a token amount, use a small positive number; zero is rejected.
- GL effect: **Dr cash (1110/1310) / Cr 3100 Owners' Equity**.
- A **50/50** two-founder split is just two `issue_shares` calls with equal
  `quantity` against the same class. Adjust quantities for other splits.
- `occurred_at` (RFC3339) backdates the issuance to the real grant date; it becomes
  the journal's book date. Omit for "now".

## SAFEs (early investors)

`record_safe(party_id, amount, currency, valuation_cap_minor?, discount_bps?)`.

A SAFE is money in now for equity later. It is booked as a **financing liability**,
**not** shares — there's no dilution and no share count until it converts at the next
priced round.

- `amount` — the investment, minor units (cents).
- `valuation_cap_minor` — the cap, minor units (optional).
- `discount_bps` — discount in **basis points**: `2000` = 20% (optional).
- GL effect: **Dr cash / Cr 2240 SAFE Financing**.
- Holder role defaults to `investor`.

## Confirm with `get_cap_table`

`get_cap_table` returns outstanding shares by holder × class with **ownership basis
points** (10,000 bps = 100%), authorized-vs-issued per class, and outstanding SAFEs.
Read it back to the founder and confirm the split looks right before moving on —
issuances post to the GL, so fixing a mistake means a correcting entry, not an edit.
