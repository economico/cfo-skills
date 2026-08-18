---
name: payment-rails
description: >
  Model how money moves in and out of a business in Economico, and pick the
  cheapest or fastest way to move it. A financial account is one pool of value
  (one GL balance); this skill registers the many identifiers that reach it — bank
  details (RFC 8905 payto), blockchain addresses (CAIP-10) — and the priced in/out
  methods on each (rail, asset, fees, ETA), plus account-level card collection.
  It also stores a counterparty's payment instructions, ranks routes
  with quote_payment_routes, and posts route fees to the ledger. Use for "register
  my Stripe / Mercury / wallet", "add ACH details / a wallet address / a payment
  rail", "set up payment routing", "cheapest / fastest way to pay this vendor",
  "which assets to accept on this invoice", "record the processor fee", or "save
  this vendor's bank / wallet details". Registering the account itself lives
  in company-setup, billing customers in invoicing, paying vendor bills in
  expense-tracking — this is the rail/route/fee layer under all three.
---

# Payment rails & routing

A **financial account** is one pool of value with one GL sub-balance — that's all
it is, and it doesn't change here (register accounts in `company-setup`). This
skill adds the layer on top: the **identifiers** that reach a pool and the
**priced ways** money moves through them, so an agent can advertise the right
assets on an invoice, pick the cheapest route to pay a vendor, and book the fee
correctly.

**Hard EVM rule:** one hex address/keypair means one endpoint, even across
chains. **Wrong:** one `add_payment_endpoint` call for Base and another for
Ethereum. **Right:** one call whose CAIP-10 `uri` uses either supported chain and
whose `methods[]` contains both routes. The URI names the keypair; each method's
CAIP-19 `asset` selects the chain and token that actually move value. Economico
enforces this: the second call is rejected as a conflict naming the endpoint that
already holds the keypair — extend that one with `update_payment_endpoint`
(existing methods **plus** the new chain's) rather than retrying with a
different URI.

## The three-level model

```
financial_account   1 pool  = 1 GL sub-balance          (company-setup owns this)
  ├── payment_endpoint   0..n identifiers                 payto:// URI or CAIP-10 address
  │     └── payment_method   1..n priced (direction, rail, asset) + fee schedule
  └── payment_method     0..n account-level               card collection (no identifier)
```

One real custodial account (Stripe's financial account is the canonical case) is
**one** `financial_account` — one balance to reconcile — with several endpoints:
its ACH virtual-account details, its address on Base, its address on Ethereum.
Registering the ACH details *and* the address as two separate financial accounts
is the mistake this model exists to prevent — that splits one real pool into two
GL sub-balances and breaks reconciliation. **One pool, many identifiers.**

- **`payment_endpoint`** — one externally-visible identifier. `uri` is either an
  RFC 8905 **payto** URI (`payto://ach/{routing}/{account}`,
  `payto://iban/DE75512108001245126199`, `payto://bic/…`) or a **CAIP-10**
  account (`eip155:8453:0xAb16…`). Owned by **either** one of our accounts
  (`financial_account_id`) **or** a counterparty (`party_id`) — never both.
- **`payment_method`** — one priceable `(direction, rail, asset)` on an endpoint,
  or attached at the account level for collection rails that put no identifier on
  the wire (`card`). `direction` is always **relative to the owner**: an
  `in` method on *our* endpoint = we can receive it; an `in` method on a
  *vendor's* endpoint = the vendor can receive it.

## 1. Register our own account's rails

Add an endpoint with its methods inline in one call:

`add_payment_endpoint(financial_account_id, uri, label?, external_ref?, methods[])`

Each method in `methods[]`:

| field | meaning |
|---|---|
| `direction` | `in` (receive) or `out` (pay) — separate rows; economics differ by direction |
| `rail` | `ach`, `ach_same_day`, `fedwire`, `rtp`, `sepa`, `sepa_instant`, `swift`, `onchain`, `internal` (free book transfer), `card` |
| `asset` | what moves on the wire — ISO 4217 (`USD`, `EUR`) or CAIP-19. For an EVM `onchain` token, use `eip155:<chain_id>/erc20:<contract_address>` (`eip155:8453/erc20:0x833…` = USDC-on-Base); the `erc20:` segment is required |
| `settles_as` | what lands in the pool if the provider converts (ACH `USD` in → `USDC` balance). Omit if same as `asset`; when set, `spread_bps` applies |
| `fee_fixed_minor` | fixed fee, minor units |
| `fee_bps` | proportional fee, basis points |
| `spread_bps` | conversion spread, bps — only when `settles_as` ≠ `asset` |
| `fee_min_minor` / `fee_max_minor` | optional floor / cap on the proportional (bps + spread) component — "0.25%, min $1, max $5" |
| `fee_variable` + `fee_estimate_minor` | network/gas — not knowable in advance; the estimate is used only for ranking |
| `min_amount_minor` / `max_amount_minor` | optional amount limits (distinct from the fee clamps) |
| `eta_seconds` | typical settlement time — the speed dimension. Instant (onchain/RTP/SEPA-instant) vs 1–3 business days (ACH) vs ~a week (SWIFT) |

**Effective cost** of moving amount X is deterministic:
`fee_fixed + clamp(X·(fee_bps + spread_bps)/10⁴, fee_min, fee_max) (+ fee_estimate if variable)`.
That plus `eta_seconds` is what `quote_payment_routes` ranks on.

**One address, many chains:** a single EVM keypair is one endpoint row; enumerate
the chains at the method level (one `onchain` method per concrete CAIP-19
chain+token). Don't create a near-duplicate endpoint per chain.

Same EVM keypair, one endpoint, priced on two chains:

```
add_payment_endpoint(
  financial_account_id = <treasury wallet>,
  uri   = "eip155:8453:0xAb16a96D359eC26a11e2C2b3d8f8B8942d5Bfcdb",
  label = "Treasury EVM address",
  methods = [
    { direction:"in",  rail:"onchain", asset:"eip155:8453/erc20:0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",  fee_variable:true, eta_seconds:15 },   // USDC on Base
    { direction:"out", rail:"onchain", asset:"eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",    fee_variable:true, fee_estimate_minor:400, eta_seconds:120 } // USDC on Ethereum
  ])
```

### Account-level card collection

`card` puts **no** payer-facing identifier on the wire — the payer never
sees one of our addresses — so they attach to the account, not an endpoint:

`add_payment_method(financial_account_id, method{...})` — rail `card` only.

- Card: `{ direction:"in", rail:"card", asset:"USD", fee_bps:290, fee_fixed_minor:30 }` — 2.9% + $0.30.

## 2. Store a counterparty's payment instructions

To pay a vendor you need where to send it. Register their
instructions as a **party-owned** endpoint:

`add_payment_endpoint(party_id, uri, label?, methods[])` — the methods carry the
vendor's receiving capability (rail, asset, `eta_seconds`); their fees are theirs,
so usually leave the fee fields empty (fill them only where **we** bear the cost,
e.g. our side of a wire).

`update_payment_endpoint(id, methods=[...])` **replaces** an endpoint's whole
method list (price sheets are current facts, not history). The `uri` is immutable
— add a new endpoint to change it. `remove_payment_endpoint` / `remove_payment_method`
soft-disable (recorded payments keep referencing them).

## 3. Quote routes

`quote_payment_routes` ranks every eligible way to move an amount — pure
computation, it never moves money:

`quote_payment_routes(direction, amount, assets?, rails?, party_id?, max_eta_seconds?, optimize?)`

- `direction` — `in` (how can we receive?) or `out` (how can we pay?).
- `optimize` — `cheapest` (default) or `fastest`.
- `party_id` — **outbound only**: intersect *our* `out` methods with that
  counterparty's stored `in` endpoints — "cheapest way to actually get funds to
  this vendor".
- `max_eta_seconds` — cheapest route that still settles by a deadline.

Each ranked route carries the account, endpoint URI, rail, asset, ETA, and the fee
breakdown. **The agent picks and then records the payment** with the chosen route.

A comparison is work in the books, not a table in the reply: register the rails
being compared first — the counterparty's instructions as a party endpoint, our
side as methods on the paying account — then quote through them. Asked to compare
without paying, still leave the endpoints on file so payment time needs no
re-typing, and never claim to have saved details that were only written in the
answer.

## 4. Post the route fee to the ledger

A route fee is real expense, not just metadata. `record_payment` (AR) and
`pay_bill` (AP) take the fee inline so the document still settles at **gross**
while cash moves at **net**:

- `record_payment(..., financial_account_id, payment_endpoint_id?, fee_amount?, fee_account_code?)` —
  cash lands **net** (`amount − fee`); journal is Dr cash net / Dr fee / Cr AR gross.
- `pay_bill(..., financial_account_id, payment_endpoint_id?, fee_amount?, fee_account_code?)` — cash out
  is **`amount + fee`**; Dr AP gross / Cr cash / Dr fee.
- `fee_account_code` picks **who** charged the fee, and the two don't mix — the
  point of the split is being able to add up a year of each:
  - **`5500` Payment Processing Fees** (the default) — a card or payment
    processor's cut of the amount (Stripe, PayPal).
  - **`6440` Bank Charges and Fees** — the **bank's** charge for moving money:
    incoming or outgoing wire fees, ACH returns, account fees. A wire fee is
    never `5500`, whichever direction the money went.
- `financial_account_id` is required — look it up with `get_financial_accounts`.
  `payment_endpoint_id` records which identifier the money moved through. **Endpoints never create sub-balances** — the GL still keys on the
  account, which is exactly what keeps one pool = one balance.

When the actual fee isn't known at record time (variable gas, a batched provider
statement), record the method's estimate and let daily reconciliation true it up
against the statement.

## Reading the picture

One read gives the full routing picture — no separate list tool:

- `get_financial_accounts` return each account's
  `endpoints` + account-level methods inline.
- `get_parties` return a party's stored payment instructions.

## Discipline

- **Read before write.** `get_financial_accounts` / `get_parties` first —
  resolve-or-create, never blind-add a duplicate endpoint. A `uri` is unique per
  business; re-registering it is rejected.
- **One pool, many identifiers.** Never register a second financial account for a
  different rail into the same real pool — add an endpoint to the existing account.
- **One EVM keypair, one endpoint across chains.** Never preserve both the Base
  and Ethereum CAIP-10 forms of the same hex address as separate endpoints.
  Pick either supported chain for the one canonical endpoint URI and put every
  chain/token in that call's `methods[]`; the CAIP-19 assets, not duplicate
  endpoints, select the networks. When adding a chain later, read the endpoint
  and use `update_payment_endpoint` with its existing methods plus the new one.
  A rejected second registration is that rule firing, not a bad URI — never work
  around it by trying another chain prefix or a second financial account.
- **Registering rails is safe metadata; moving money is not.** `add_payment_endpoint`
  / `add_payment_method` / `quote_payment_routes` don't touch the GL. `record_payment`
  / `pay_bill` move cash and post the fee leg — confirm those with the user.
- **Units are minor (cents) and basis points.** `2.9% → fee_bps:290`; `$0.30 →
  fee_fixed_minor:30`; `$5 cap → fee_max_minor:500`.
- **Assets:** ISO 4217 or CAIP-19 only; for `onchain`, CAIP-19 (it pins chain +
  token). The books are USD-only — non-USD assets that don't fold to USD can't
  settle a payment.
- **Price sheets are current facts.** When a provider reprices, `update_payment_endpoint`
  to **replace** the methods; don't accumulate stale rows.

## Hand offs

- Registering the financial account itself, and first-run setup → **`company-setup`**.
- Billing a customer / recording their payment (with the fee leg) → **`invoicing`**.
- Recording & paying vendor bills (with the fee leg), and capturing vendor
  payment instructions in context → **`expense-tracking`**.
- Reading the books → **`financial-analyst`**.
