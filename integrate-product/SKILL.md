---
name: integrate-product
description: "Integrate a founder's application backend with Economico over REST or MCP so sign-ups, plan changes, cancellations and live product usage record cleanly into the books."
---

# Integrate a product backend with Economico

Follow [economico](../economico/SKILL.md) for ground rules. Rehearse every integration against a
disposable sandbox business before pointing at real books. For pricing design or founder margin
reviews, use [product-economics](../product-economics/SKILL.md).

## Integration architecture

1. **Authenticate as a headless client:** Register a confidential client public key (RS256 >= 2048
   bits or ES256/P-256) on the target business and mint short-lived tokens with `private_key_jwt`
   at `/oauth/headless/token` (see [connect: REST and headless agents](../economico/references/connect.md#rest-for-a-product-backend)).
   Grant the narrowest permissions the backend actually needs.
2. **Discover schemas from OpenAPI or `catalog`:** Call `GET /v1/openapi.json` with
   `X-Economico-Business: <slug>` (or `catalog {action: "list"|"describe"}` over MCP) for the
   permission-filtered command and read surface. Headless tokens are refused `interactive` and
   `browser` commands.
3. **Establish reusable templates once:** Save the product's plans as `on_sale` templates with
   `templates.create` (see [modeling products](../product-economics/references/modeling-products.md))
   and pin their `documentId`s in configuration rather than re-creating rights on every sign-up.
4. **Execute commands idempotently over REST:** Send `POST /v1/commands` with
   `X-Economico-Business: <slug>` and the command envelope (`name`, `input`, UTC `effective_at`,
   and a deterministic `idempotency_key` derived from the app's own event/checkout/webhook ID).
   - **Retry safety:** Retrying the exact `idempotency_key` replays the original receipt; reusing a
     key with different input is refused; domain no-ops answer `code: "no_change"`.
   - **Provenance:** Store order/checkout evidence via `documents.receive` with the source ID as
     `externalId`, and copy source event IDs unchanged into `sourceFactId` (never reorder or suffix
     without preserving the source event).
5. **Map product lifecycle events to Economico commands:**
   - **Sign-up:** `parties.create` (`cus_<id>`) → `contracts.create` (from the plan template) →
     `contracts.accept` with checkout/order evidence.
   - **Usage / delivery:** `contracts.record` (`usage` / `delivery` / `recognition` / `billing`
     phases) on the matching `obligationKey` with integer minor units or scaled integer quantities
     matching the right definition.
   - **Plan / seat change or cancel:** `contracts.amend`, `contracts.change_quantity`,
     `contracts.cancel`, or `contracts.terminate` with `expectedDocumentId` and `expectedStatus`.
   - **Payments:** Record settled payments separately via `payments.record` (or Stripe settlement
     workflows — see [explore Stripe](../manage-sales/references/explore-stripe.md)) against open
     claims read from `POST /v1/reports` (`kind: "activity_claims"`).

## References and recipes

- [Connect (REST & headless auth)](../economico/references/connect.md#rest-for-a-product-backend)
- [How Economico works](../economico/references/how-economico-works.md) — command envelope,
  idempotency, terms vs. facts, refusals
- [Modeling products](../product-economics/references/modeling-products.md) — reusable templates
  and `priced-rights-v1` rights
- Fetch working command sequences via `catalog {action: "recipe", name}`:
  `customer-monthly-subscription`, `customer-per-seat-subscription`,
  `customer-platform-fee-plus-usage`, `customer-prepaid-credits`,
  `customer-included-usage-overage`
