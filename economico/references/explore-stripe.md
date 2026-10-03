# Explore Stripe

Stripe knows what was charged, to whom, and when. It does not know what was agreed. Use it as
evidence to reconcile against signed terms and the product's code, never as the authority over
them. Economico has no Stripe connection: you read Stripe with the founder's access and record
what you learned.

## Access

Use whatever the founder grants, read-only:

- the Stripe MCP server, if the founder connected it to your host. It can also write: call only
  its read tools;
- the Stripe CLI after the founder runs `stripe login`, or with a restricted read-only key the
  founder puts in the `STRIPE_API_KEY` environment variable. Never type, paste or echo a key
  into a command yourself: command lines end up in shell history, process lists and
  transcripts;
- an export the founder drops in the workspace (CSV or JSON).

Never create, update or refund anything in Stripe. If only test mode is available, say so: test
data describes the integration, not the business.

## What to pull

```bash
stripe products list --limit 100 --active true
stripe prices list --limit 100 --expand data.tiers
stripe subscriptions list --status all --limit 100 --expand data.customer
stripe customers list --limit 100
stripe invoices list --limit 100 --status paid          # and --status open
stripe billing meters list                              # usage-based billing
stripe balance_transactions list --limit 100            # fees, payouts, refunds
stripe payouts list --limit 100
```

Page with `--starting-after <last id>` until `has_more` is false.

## Map Stripe objects to the model

| Stripe | Becomes | Notes |
|---|---|---|
| Product + its active recurring Price(s) | One template per purchasable plan (`plan: "on_sale"`) | Record the product and price ids in the activity's `externalReferences` |
| Price `recurring.interval` month / year | Monthly or annual schedule | An annual price billed upfront is the annual-prepaid recipe |
| Price `billing_scheme: tiered`, `tiers_mode` | A `tiered` calculation, `graduated` or `volume` | Copy tier boundaries and unit amounts exactly |
| Price `recurring.usage_type: metered`, a billing meter | A usage activity with a `rate` or `tiered` calculation | The meter's event name is the usage unit |
| Price `transform_quantity` | `unitsPerPrice` | "$2 per 1,000" is rate `200` with `unitsPerPrice` `1000` |
| `unit_amount_decimal` (on a price or a tier) | Rate over a larger unit | Stripe states it in **cents**, like `unit_amount`: `"0.002"` is 0.002¢ a unit (a fiftieth of a cent), not $0.002. Shift the decimal point off the rate and onto `unitsPerPrice`: `"0.002"` is rate `2` per `1000` units, `"0.15"` rate `15` per `100`. Never round the rate to a whole cent a unit |
| Subscription (active, trialing, past_due, or canceled inside the history window) | One customer contract bound to the plan's template | Contract terms `start` = the subscription's `start_date` when you record history, so every past period is billed and earned; the current period's start only when the founder wants no history. A canceled one ends at `ended_at` |
| Subscription `quantity` > 1 | Seats: price × quantity | Use a seat template or a per-contract price term |
| Coupon / discount on a subscription | A per-contract price term | Keep the list price in the template; the discount is this customer's term |
| Customer | A party with `categories: ["customer"]` | Party id from the Stripe id: `cus_…` is already a good id |
| Paid invoice | Evidence for a bill occurrence, and the charge for its `collect` | `sourceFactId` `stripe:invoice:<in_id>` |
| Charge / payment intent | Payment evidence: `{ "kind": "stripe_object", "ref": "ch_…" }` | |
| Balance transaction fee | Payment processing expense on 5230 | Stripe is a vendor: see [vendor contracts](vendor-contracts.md) |
| Payout | A transfer between the Stripe balance and the bank | Register the Stripe balance as its own financial account if you record cash at Stripe |
| Refund | A linked credit or refund on the original contract | Record it against the original invoice's claim |

## Reconciling

- For each active subscription, check the price against the plan in code. A price that no plan
  uses is a legacy or negotiated price: model it as a per-contract term and flag it.
- For each customer, look for a signed agreement in email. If one exists, the agreement's terms
  win and Stripe is evidence of billing under it.
- Sum MRR in code from the subscriptions you will record (price × quantity, annual ÷ 12, minus
  recurring discounts). Compare with the MRR Stripe's dashboard shows, if the founder can tell
  you. Explain any difference before recording.
- Stripe's currency and amounts are already minor units, the decimal ones too. A USD
  `unit_amount` of 4900 is `"4900"` in Economico, and a `unit_amount_decimal` of `"0.002"` is
  rate `2` per `1000` units. Before recording a usage price, price one round quantity both ways
  (Stripe's decimal × quantity in cents, and `activity_effects` or the rate × quantity ÷
  `unitsPerPrice`); a factor of 100 between them is a dollars-for-cents slip.

## Cash: gross or net

Customers pay the gross amount; Stripe deposits the net after fees. Keep the three facts
apart, because each settles something different:

1. **The charge**: each invoice's `collect` at gross, into a financial account for the Stripe
   balance. This is what clears the customer's receivable; one charge settles one invoice.
2. **The fee**: Stripe's fee as a vendor expense on 5230 payment processing, paid out of the
   Stripe balance account (Stripe is a vendor contract like any other).
3. **The payout**: a transfer of the net from the Stripe balance account to the bank.

Record them in that order, in date order: every customer receipt into the Stripe balance account
first, then the Stripe vendor contract's fees and payouts. A payout is a `transfer_cash` effect
on the Stripe vendor contract (`financialAccountFact` naming the Stripe balance account,
`toFinancialAccountFact` the bank), and it is refused unless the Stripe balance already holds
the net it moves; a contract's recordings must go forward in time, so a payout recorded before
the receipts behind it cannot be fixed by re-dating.

There is no shortcut that records only payouts: a payout is a net sum across many invoices, so
recording it alone leaves every invoice open in receivables, and recording the fee against the
bank as well takes it off cash twice. Cash at the bank then equals the payouts, and the Stripe
balance account returns to zero after each payout.
[stripe-gross-net-payout](recipes/stripe-gross-net-payout.json) is the whole sequence, tested: two
charges at gross, their fees, one payout of the net, and a payout too large for the balance
refused.
