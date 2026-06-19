# Researching a vendor to build its spine

The point of the contract + obligation spine is that it captures *what we're
paying for and why*, not just an amount. A receipt alone rarely says enough — so
research the vendor before (or while) you record the bill. Spend research effort
in proportion to the spend: a $9/mo tool needs a glance; a $4k/mo infra bill is
worth getting right.

## Sources, in order of usefulness

1. **The receipt/invoice itself** — line items, plan name, billing period,
   invoice number, the ToS/"terms" link and a "manage subscription" link in the
   footer. This is your primary source.
2. **The signup / welcome email** — search the inbox for the vendor's onboarding
   message. It names the plan you chose and usually links the pricing page.
3. **The vendor's pricing page** — fetch it to confirm the plan, the cadence
   (monthly/annual), per-unit/metered pricing, and what each product is. Use it
   for `order_form_url` and to choose the account code + SKU.
4. **The vendor's terms-of-service page** — use its URL for the contract `msa_url`
   (vendor contracts accept the vendor's own terms). You don't need to read it in
   full; the URL records which terms the spend is under.

## What the research feeds

| Research finding | Where it goes |
|---|---|
| Vendor website / domain | `create_party(url=…)` |
| Terms-of-service URL | `create_contract(msa_url=…)` |
| Pricing page URL | `create_contract(order_form_url=…)` |
| Plan / cadence / scope | `services_scope`, `term_length`, `payment_terms` |
| What each product is | obligation `name`, `account_code`, `sku` |
| Per-unit / metered pricing | obligation `type="usage"`, `metric_name`, `price_per_unit_minor` |
| Subscription price + cadence | obligation `type="recurring"`, `amount_minor_units`, `interval` |

## Picking the obligation type

- A flat monthly/annual subscription → `recurring` with `amount_minor_units` +
  `interval`.
- Metered / pay-as-you-go (per-request, per-seat-overage, per-GB) → `usage` with
  `metric_name` + `price_per_unit_minor` (no `amount_minor_units`).
- A one-time charge (setup fee, hardware) → `one_off` with `amount_minor_units`.

## When you can't research (no network, ambiguous receipt)

Record the bill anyway so the books stay current: create the party, an active
vendor contract with whatever URLs you have (or none), and a single obligation
with your best-guess account code (default 6350 SaaS or 5900 COGS). Note in the
bill `memo` what you couldn't confirm so it can be refined later — an approximately
categorized bill beats an unrecorded one.
