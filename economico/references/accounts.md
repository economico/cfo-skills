# Choosing accounts

Every business shares one code-owned chart. An effect pattern fixes the account pair; you choose
only the revenue or expense leaf and, for expenses, the function. There is no chart-listing read:
this page is the working subset. Codes remembered from older Economico material (5300 hosting,
5400 inference, 6xxx operating expenses) have changed meaning or no longer exist; use these.

## Revenue

| Code | Account | Use for | Category that selects it |
|---|---|---|---|
| `4110` | Subscription revenue | Recurring plan fees, seats | `subscription` |
| `4120` | Usage-based revenue | Metered usage, overage, credits consumed | `usage` |
| `4100` | Product sales revenue | One-off product sales | `services` (the default) |
| `4200` | Service revenue | Implementation, onboarding, setup fees | set `revenueAccount` |
| `4210` | Consulting revenue | Hourly, retainer and fixed-fee consulting | `consulting` |
| `4130` | Licensing revenue | Licences | set `revenueAccount` |
| `4220` | Platform and marketplace revenue | Take rate on a marketplace | set `revenueAccount` |
| `4500` | Grant and award income | Grants | `grant` |

Deferred revenue is `2150`, receivables `1120`, unbilled usage `1125`; the patterns post those.

## Expenses: nature plus function

An expense account says what was bought; the `expenseFunction` says what it was for:
`cost_of_revenue`, `research_and_development`, `sales_and_marketing`,
`general_and_administrative`, `other`. Gross margin reads cost of revenue, so this choice matters
more than the account.

| Typical vendor | Code | Account | Usual function |
|---|---|---|---|
| AWS, GCP, Azure, Cloudflare, Render, Fly, Vercel, Neon, Supabase (production) | `5210` | Hosting and cloud infrastructure | `cost_of_revenue` |
| The same, for staging, CI or development | `5210` | Hosting and cloud infrastructure | `research_and_development` |
| OpenAI, Anthropic, model APIs called by the product | `5220` | AI and model inference | `cost_of_revenue` |
| Stripe, Paddle, PayPal fees | `5230` | Payment processing fees | `cost_of_revenue` |
| Twilio, Resend, Postmark, Algolia, data vendors used by the product | `5240` | Third-party APIs and data | `cost_of_revenue` |
| Notion, Google Workspace, Slack, Linear, 1Password | `5310` | SaaS and productivity tools | `general_and_administrative` |
| GitHub, Sentry, Cursor, Claude or ChatGPT subscriptions for the team | `5320` | AI and developer tools | `research_and_development` |
| Contractors, employer-of-record | `5140` | Contractors and EOR services | by the work: `research_and_development` for engineering |
| Customer support tools and outsourced support | `5150` | Customer support services | `cost_of_revenue` |
| Ads, sponsorships | `5410` | Advertising and paid media | `sales_and_marketing` |
| Conferences | `5420` | Events and conferences | `sales_and_marketing` |
| Lawyers, incorporation | `5620` | Legal and corporate fees | `general_and_administrative` |
| Accountants, tax preparers | `5630` | Accounting and tax services | `general_and_administrative` |
| Insurance | `5530` | Insurance | `general_and_administrative` |
| Domains, registrars | `5310` | SaaS and productivity tools | `general_and_administrative` |
| Travel | `5710` | Travel | by purpose |
| Bank fees | `5910` | Bank charges and fees | `general_and_administrative` |
| Anything else | `5990` | Other operating expense | ask |

Set both in the effect: `"expenseAccount": 5220, "expenseFunction": "cost_of_revenue"`. Without
them the effect's `category` picks a default (`subscription` → 5310 general and administrative,
`usage` → 5220 cost of revenue, `consulting` → 5140 cost of revenue, `services` → 5990 cost of
revenue), which is rarely what a specific vendor needs.

Classify by environment and purpose, not by vendor name: the same cloud bill can serve production
(cost of revenue) and development (research and development); split it into two activities when
the invoice separates them. When a choice changes gross margin materially and the evidence is
ambiguous, ask the founder and explain the effect.

## Balance-sheet accounts the recipes touch

`1110` cash (per registered bank account), `2190` credit card payable (per registered card),
`2110` payables (per vendor), `2135` due to related parties (per founder), `1150` prepaid
expenses, `1170` recoverable input tax, `2160` output tax, `2240` SAFE and other financing,
`3100`/`3110` share capital and premium.
