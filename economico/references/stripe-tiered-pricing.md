# Stripe tiered pricing

A tier schedule is the price calculation of one right. Compose that right with separate setup,
access or other service rights in a reusable plan or inline client contract using
[business model patterns](business-model-patterns.md). Never turn each price band into a service
or make a tier's conditional flat fee an unconditional package charge.

## Read and map the source

Retrieve the Stripe Price with its expanded tiers and the subscription item or meter supplying
quantity. Preserve its Price ID and supporting agreement. Distinguish the agreed licensed
quantity from aggregated metered usage; don't guess the meter's aggregation or billing period.
Read the live activity schema before constructing the definition.

| Stripe source | Priced-right calculation |
|---|---|
| `billing_scheme: tiered` | `type: "tiered"`, with quantity referencing the agreed term or measured fact |
| `tiers_mode: volume` | `mode: "volume"`: select the tier containing total quantity, charge that tier's rate on all units and its flat fee once |
| `tiers_mode: graduated` | `mode: "graduated"`: charge each reached band's units at its rate, adding that band's flat fee once |
| Each finite `up_to` | Inclusive `upTo` reference, in the quantity's declared unit; preserve strictly increasing boundaries |
| Final `up_to: inf` | Final unbounded tier: omit `upTo` |
| Tier `unit_amount` | Integer monetary `rate` term; use zero for a flat-only tier |
| Tier `unit_amount_decimal` | Exact integer rate and `unitsPerPrice` denominator; preserve fractional minor units, never round the rate first |
| Tier `flat_amount` | Integer monetary `flat` term on that tier; omit or use zero when absent |

A tier can contain a unit rate, a flat fee, or both. Rates and flat fees are in the source
currency's minor units, not major units. For example, a decimal rate of `"0.002"` cents maps to
rate `"2"` USD minor units per `"1000"` quantity units. Default `unitsPerPrice` is one.
Do not enable `wholeUnits` or `startedUnits` for ordinary Stripe tiers: they round partial
pricing blocks. Stripe package pricing on a non-tiered price (`transform_quantity` with
`divide_by` N) is one unbounded tier with `unitsPerPrice` N: `round: "up"` is
`startedUnits: true` (every started package charged in full), `round: "down"` is
`wholeUnits: true`.
Do not apply `transform_quantity` as a mere division: its rounding changes quantity, and Stripe
cannot combine it with tiered pricing. Retain unsupported transformations as a gap.
Parse `flat_amount_decimal` exactly: normalize whole minor-unit values such as `"1000.0"`
to `"1000"`. If a source supplies a genuinely fractional minor-unit flat amount, the current
integer `flat` term cannot represent it; preserve the gap rather than silently rounding it.

## Check the boundaries before saving

Compute the source-backed charge at zero, each finite boundary and its next unit, and a quantity
in the final tier. Include decimal rates and every flat fee. Stripe charges the first tier's
flat fee even at zero quantity in both modes. If the source has a first-unit workaround to waive
that charge, preserve the actual tiers rather than changing zero handling globally. Volume
pricing can decrease the total at a boundary; don't impose monotonic totals.

The kernel sums graduated rational unit charges before rounding half up once to the currency's
minor unit, with integer flat fees. Compare fractional-rate results against the source invoice
or Stripe preview before claiming parity with that Price; a rounding difference is a gap, not
a license to adjust a rate. Currency, tax, discounts, proration and usage aggregation must also
match the source; tier arithmetic alone does not establish complete invoice parity.

After saving, read back the right's tier schedule and its template/contract bindings. For a
plan-only request, stop at the saved plan; for a draft-only request, stop at the draft. When
recording actual billed usage is requested, use one evidenced period quantity and obligation,
then verify the resulting claim; do not replay each tier as a separate usage occurrence.

## Validation reference

The repository's package tests run both modes through public commands, with saved templates
and inline contracts, alongside separate setup and access rights. They cover the unit-rate,
unit-plus-flat, flat-only, free-first-band and zero-usage-waiver shapes, including boundaries and a decreasing
volume total. The documented 12-unit example yields $66.00 volume or $111.00 graduated before
separate package charges. These are test examples, not default prices to put in a business.

Source: [Stripe tiered pricing](https://docs.stripe.com/subscriptions/pricing-models/tiered-pricing)
and [Price creation](https://docs.stripe.com/api/prices/create).
