# Explore the codebase

The product's own code is the most honest description of the business: it enforces who gets
what, it meters what is billed, and it calls the services that cost money. Read it before Stripe
and before email, so you know what you are looking for there.

Read only. Do not run migrations, deploy, or call production APIs. Never copy a secret into the
brief, a command or a document.

## What you are looking for

| Question | Where it usually lives | Search for |
|---|---|---|
| What does the product do, for whom? | `README`, landing page source, `docs/`, marketing site, `package.json` description | product name, "pricing", "for teams", hero copy |
| What plans exist, with which limits? | a plans/tiers/entitlements module, feature flags, DB enums, seed data | `plan`, `tier`, `pro`, `enterprise`, `entitlement`, `limit`, `quota`, `seats`, `features` |
| What does it charge, and how? | billing module, Stripe calls, checkout pages, webhooks | `stripe.`, `price_`, `prod_`, `checkout.sessions`, `subscriptions`, `invoice`, `usage_record`, `meter_event`, `billing_meter` |
| What is metered? | usage tracking, rate limiting, event counters | `usage`, `meter`, `increment`, `track`, `credits`, `tokens`, `generations`, `requests` |
| When is a customer "active"? | webhook handlers | `customer.subscription.created`, `invoice.paid`, `checkout.session.completed`, `trial` |
| What does it run on? | infra config, deploy scripts, env examples | `wrangler.toml`, `vercel.json`, `fly.toml`, `render.yaml`, `Dockerfile`, `terraform/`, `serverless.yml`, `.env.example` |
| Which paid APIs does it call? | dependencies, clients | `package.json`, `requirements.txt`, `go.mod`: `openai`, `@anthropic-ai/sdk`, `resend`, `postmark`, `twilio`, `algolia`, `pinecone`, `sentry`, `posthog` |
| Is there anything unusual? | | marketplace fees, revenue share, referral payouts, grants, crypto payments (`x402`, `usdc`) |

`.env.example` (never `.env`) is the fastest vendor inventory: every `*_API_KEY` names a service
someone pays for. Dependencies name the rest.

## Turn findings into model inputs

For each plan, write down:

- **name and identifier** as the code knows it (`pro`, `team`) and, if referenced, the Stripe
  price id;
- **what the customer gets**: features, included quantities (per month? rolling?), seat limits;
- **how it is priced**: flat, per seat, per unit of usage, tiered, prepaid credits, annual option;
- **what happens at the limit**: hard stop (no overage revenue), overage billed, or upgrade prompt;
- **the usage unit and where it is counted**: the table, event or counter the product already
  keeps. That is the future source of usage facts.

For each service the product depends on, write down: the vendor, what it is for, whether it
serves customers in production (cost of revenue) or the team (research and development, or
general and administrative), how its cost scales (flat, per seat, per usage), and which
product actions call it ("generate text" calls the model API and the hosting): those are the
vendor's `drivers` in the [model preview](business-model-brief.md#preview-the-model-before-recording-it).

Classify by environment, not by vendor: production hosting is cost of revenue; a staging
environment or CI is research and development. Say which you assumed.

## Pitfalls

- Marketing copy and code disagree. The code's limits are what customers actually get; the
  marketing page is what they were promised. Note both.
- Dead plans survive in code for grandfathered customers. They are still templates if anyone is
  on them (Stripe will tell you).
- A free tier is not revenue and needs no template unless it converts to a paid plan inside a
  contract (a trial). Mention it in the brief.
- Do not infer prices from test fixtures or seed data unless nothing else exists; mark them as
  such.
