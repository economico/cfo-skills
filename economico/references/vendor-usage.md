# Vendor usage and the subscription review

Which subscriptions are used, by whom, and what each costs. The agent brings the facts from the
vendor's own reports; Economico holds no standing vendor credentials. For recording priced meters
and reconciling bills, see [fetch and record detailed vendor usage](vendor-contracts.md#fetch-and-record-detailed-vendor-usage).

## How often to record

Every observation batch counts against the plan's event allowance, so batch. One
`contracts.observe_usage` call takes up to 200 observations and writes one event per usage period.

- **Seat and platform use** ("this seat was active", "we used this tool"): once a billing cycle,
  usually monthly. What matters is knowing month by month which tools are in use. A batch per vendor
  per month is enough.
- **Metered usage** (tokens, CI minutes, requests): your judgment, from immediately to monthly. Go
  more often when the cost moves fast, a credit or allowance is close to running out, or the vendor's
  data expires soon. Go less often when the amounts are small and steady.

Vendor data expires, so fetch before it does, whatever the cadence.

## Record seat and platform use

A plan's meters cover what the vendor bills. To count use, add a meter to the vendor's existing
contract once, with `contracts.add_usage_meter`: `key`, `name`, `unit` (such as `seat` or
`active day`), `effectiveAt` and evidence of where the usage comes from. It costs nothing, posts
nothing and is never billed. Don't open a second contract for it. Date it the first of a month when
you can, since a later backdated amendment can't precede it. A contract with observations can't be
voided, so add the meter once the contract is settled.

Then each cycle, preserve the report with `documents.receive` and call `contracts.observe_usage`
with one observation per holder: `quantity` is `"1"` for an active seat, or the count the report
gives, and `"0"` when the report you fetched shows nobody used it, so that reads as no use rather
than no report; `attribution.holder` is who holds the seat, a pseudonym or an email address, as the business
gives it. Events can't be erased, so an email recorded as a holder is permanent; a business that
wants it erasable gives a pseudonym it maps to the person elsewhere. `sourceId` joins the period and
holder, so fetching again is safe. For a tool with no per-person view, record one observation for
the tool.

## Where the usage is

Ask for the narrowest access that works, and use the business's own credentials. Never store a key.

| Vendor | Source | Access |
|---|---|---|
| GitHub | billing usage API (`/organizations/{org}/settings/billing/usage`, `/usage/summary`), metered CSV reports | organization owner or billing manager, "Administration" read |
| OpenAI API | Administration API: `/v1/organization/usage/*` for quantities, `/v1/organization/costs` | Admin API key with `api.usage.read`; ChatGPT has a separate surface |
| Anthropic API | Admin API: `/v1/organizations/usage_report/messages`, `/v1/organizations/cost_report` | Admin API key; each way of buying has its own surface |
| Cursor | Admin API, per member usage events | team admin key; the docs conflict on which plans have it |
| Vercel | FOCUS charges, `GET /v1/billing/charges`, daily | team token, Pro or above |
| AWS | CUR 2.0 through Data Exports, delivered to the business's own S3 bucket | the business's IAM credentials |
| Google Workspace | Directory API `users` (`lastLoginTime`), License Manager | admin read; it gives use, not cost |
| Slack | `team.billableInfo`, active per member | Owner or Admin token |
| Notion | `List users` counts seats but not activity; last active is Enterprise only | integration token |

For anything else, look for an admin usage page or API, and take an export from the founder if
there's no API. With no usage source, card spend and sign-in logs tell part of it: say "no sign-in
seen", naming the source, not "unused".

## The subscription review

Run this on request or in a bookkeeping session. Start from the `vendor_utilization` report for the
months under review, which gives each vendor contract's spend, last use, seats and findings; pass
`unused_days` if the founder set another threshold. Check its findings with the steps below before
you present them, and say what you read. A count fetched partway through a month reads as that whole
month, so re-fetch the finished month before reporting seats as unused. Its spend is recognized cost, so a bill not yet recognized
shows none, and trials need the founder's notes or the vendor's emails.

1. List the vendor contracts with `subjects`. On each, `activities[].definition.usage` says whether a
   usage meter exists at all, which separates "no usage source" from "no use".
2. **Cost.** Read what was billed from `activity_claims` for the contract, and for metered rights the
   gross and credit in `usage_estimates`. Don't take cost from `activity_balances`: it can show prepaid
   and payable lines rather than the month's expense. It includes metered actuals and payments at
   contract level, without splitting a shared invoice into individual rights; use the usage reports
   for period and right detail.
3. **Use.** Read `usage_estimates` for the months under review, one contract at a time. For holders,
   `usage_cost` with `group_by: "holder"` covers the whole business a page at a time: filter its rows
   on `contractId`.
4. **Seats and price.** Seats are `quantities.<right>.current` on the contract when it has one, since
   the term goes stale after a seat change; otherwise the right's seat term. The report names no saving:
   price unused seats from the right's own terms, and convert to a month if its period is longer. If
   neither states a count, the count is unknown.
5. **Dates.** Read `contract_deadlines` for the next 60 days. It holds contracts with lifecycle terms
   only, and doesn't know trials: take those from the founder's notes or the vendor's emails.
6. Report only what those reads show, with the saving per item:
   - **Unused:** spend, a report you fetched for the period, and no use in the last 60 days, unless the
     founder set another threshold. State the threshold you used.
   - **Over-provisioned:** paid seats against active holders. The saving is the unused seats at the
     seat price, stated per month.
   - **About to change:** a trial converting, a renewal or a term ending.
   - **Unknown:** spend with no usage source, or a meter whose fetching lapsed. List it apart: it isn't
     a finding, and no row in a report doesn't mean no use.
7. Say what you read and for which months. Never invent a finding.

Propose the change and leave it to the founder: cancel, downgrade or remove seats at the vendor. Once
they confirm, record the amended or ended contract with the vendor's confirmation as evidence. If a
tool runs on credit, compare the cost before credit: credit doesn't make a tool cheap.
