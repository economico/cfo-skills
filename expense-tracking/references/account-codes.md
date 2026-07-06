# Expense account codes (vendor obligations)

Vendor obligations must carry an **expense/COGS** account code — a 5xxx
(cost of revenue) or 6xxx (operating expense) code. This is the single most
important judgement call in the spine: it decides where the spend lands on the
P&L and whether it counts against gross margin (COGS) or operating margin (opex).

`list_chart_of_accounts(currency="USD")` is always the authoritative, live list —
read it when unsure rather than trusting this table. Pick the code from the
receipt + the vendor's pricing page (what is this spend *for*?), and fall back to
the defaults below when the category is genuinely unclear.

## Cost of revenue — 5xxx (counts against gross margin)

Use these when the spend is a direct cost of delivering the product to customers.

| Code | Name | SKU-keyed? | Typical vendors |
|------|------|-----------|-----------------|
| 5100 | Direct Salaries and Wages | no | payroll for delivery staff |
| 5200 | Direct Contractors and EOR | no | delivery contractors, EOR |
| **5300** | Hosting and Infrastructure | **yes** | AWS, GCP, Cloudflare, Vercel, Fly.io |
| **5400** | AI and LLM Inference Costs | **yes** | OpenAI/Anthropic API, Replicate, inference |
| 5500 | Payment Processing Fees | no | Stripe, PayPal, processor fees |
| **5600** | Third-Party API and Data Costs | **yes** | Twilio, data/API subscriptions in the product |
| 5700 | Customer Support Costs | no | support tooling/staff |
| **5900** | Other Cost of Revenue | **yes** | COGS that doesn't fit above (**default COGS**) |

## Operating expenses — 6xxx (below gross margin)

Use these when the spend runs the company rather than delivering the product.

- **R&D**: 6130 R&D Cloud and Dev Tools (GitHub, CI, dev AWS), 6140 R&D AI/LLM
  Tools (ChatGPT Plus, Claude Pro), 6150 R&D Software Licenses.
- **Sales & Marketing**: 6230 Advertising and Paid Media, 6240 Marketing Tools
  and Software (HubSpot, Mailchimp), 6250 Events and Conferences,
  6270 Sales Tools and Commissions.
- **G&A**: **6350 SaaS and Productivity Tools** (Slack, Notion, Google Workspace,
  Zoom, 1Password — **default opex for general software**), 6330 Rent and
  Facilities, 6360 Insurance, 6370 Professional Fees, 6380 Legal and Corporate,
  6390 Accounting and Tax, 6420 Equipment and Hardware, 6430 Travel and
  Entertainment, 6440 Bank Charges and Fees.

## Split infrastructure vendors by environment

The same infra vendor often serves both production and development. Classify by
environment, not by vendor: the production footprint serving customers → **5300**
(COGS); dev/staging/CI environments → **6130 R&D Cloud and Dev Tools** (opex).
Model them as separate obligations on the same contract (e.g. a `prod-compute`
obligation on 5300 and a `dev-compute` obligation on 6130) so gross margin
carries only customer-serving cost. Usage lines (egress, bandwidth) default to
5300 when the traffic is overwhelmingly production. Where the vendor has a CLI
or console, read the actual running footprint and per-resource list prices to
size each environment's line instead of estimating one blended amount from the
last bill.

## SKU sub-keys (unit economics)

Only the COGS accounts **5300, 5400, 5600, 5900** (and revenue accounts) carry a
SKU sub-key. On those, set the obligation's `sku` to the specific product so spend
aggregates per unit (e.g. `sku="ec2"` / `sku="s3"` on 5300; `sku="gpt-4o"` on
5400). On non-SKU accounts (5100/5200/5500/5700, all 6xxx) the SKU is ignored —
don't bother setting it.

## Defaults when unsure

- Software that helps *deliver the product to customers* → the matching 5xxx COGS
  code (5300/5400/5600), with a SKU. If it's COGS but you can't place it → **5900**.
- General business software / SaaS → **6350**.
- Pick the most specific code you can defend from the receipt + pricing page; note
  your reasoning in the bill `memo` so a human can re-categorize if needed.

## Customer-revenue link

If a vendor cost exists specifically to serve a customer revenue obligation (e.g.
dedicated infra for one customer), set the vendor obligation's
`source_obligation_id` to that customer obligation. This is what lets
`financial-analyst` compute true gross margin per customer/SKU.
