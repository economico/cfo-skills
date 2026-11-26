---
name: product-economics
description: "Design and review SaaS pricing tiers, reusable product plans, rights templates, metered or graduated pricing, vendor-catalog publishing, product margins and unit economics (cost to serve) in Economico."
---

# Product economics and pricing

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; fetch recipes via `catalog {action: "recipe", name}`; confirm the target business
before writing). For wiring a product's backend code to Economico's API, use
[integrate-product](../integrate-product/SKILL.md).

## What this skill owns

1. **Model what you sell:** Turn code, Stripe prices or pricing pages into reusable
   `priced-rights-v1` rights and `templates.create` plans (`on_sale` and `off_sale`), keeping
   included, free and promotional rights visible. Read
   [modeling products](references/modeling-products.md) and
   [business model patterns](references/business-model-patterns.md).
2. **Tiered and metered pricing:** Map volume/graduated tiers, allowances, included usage + overage,
   and prepaid credit lots accurately. Read
   [Stripe tiered pricing](references/stripe-tiered-pricing.md).
3. **Publish plans or patterns:** Preview and publish `on_sale` templates or patterns to the public
   vendor catalog with `plan_publications` when the founder asks.
4. **Review margins and unit economics:** Pair revenue from the books (`revenue_summary`,
   `saas_metrics`, `party_contract_values`) with cost-to-serve reads (`unit_economics`,
   `usage_cost`) and vendor utilization (see
   [vendor usage](../manage-vendors/references/vendor-usage.md)). Keep unattributed costs visible;
   never invent allocations.

## References and recipes

- [Modeling products](references/modeling-products.md) — rights, templates, capacity lots,
  publishing to the vendor catalog
- [Business model patterns](references/business-model-patterns.md) — discovering reusable patterns,
  combining offerings, bespoke drafts
- [Stripe tiered pricing](references/stripe-tiered-pricing.md) — mapping Stripe volume and
  graduated tiers
- Shared foundations in `economico`: [accounts](../economico/references/accounts.md),
  [explore codebase](../economico/references/explore-codebase.md),
  [verify](../economico/references/verify.md), [show the founder](../economico/references/show-the-founder.md)
- Fetch recipes via `catalog {action: "recipe", name}`: `customer-monthly-subscription`,
  `customer-annual-prepaid`, `customer-per-seat-subscription`, `customer-platform-fee-plus-usage`,
  `customer-prepaid-credits`, `customer-included-usage-overage`, `customer-consulting-hourly-milestone`
