# People and provider payroll

Economico records who works with the business and the evidenced results of a payroll provider.
It links those results to People, accounts and independent payments. The provider calculates
pay, deductions and taxes, files returns and pays workers. Economico records those facts.
Use this workflow for US PEOs, international EORs, ordinary payroll processors and direct
contractors. Deel, Remote, Rippling PEO, Justworks PEO and Gusto payroll are starting cases;
another provider uses the same commands and evidence requirements.

## 1. Discover the provider and source facts

Use the provider's API, CLI or MCP already connected to the founder's agent, or supplied exports
and documents. Discover the installed tool's schema and granted scopes; do not assume access
because a public API exists. No Economico provider credentials or dedicated connector are needed.
Identify the exact product, provider account, actual invoicing entity, legal employer, workers,
currency, work periods and accepted service agreement. Store source evidence with
`documents.receive`; retain stable account/document/line identifiers. Keep payroll secrets,
bank details, government IDs and unnecessary payslip details out of party names and identifiers.

Read the live catalog before every command. Search the vendor catalog first and adopt a fitting
service-fee plan under the actual seller. The provider's service fee is separate from gross pay,
employer burden, benefits and funding. Fee-plan adoption does not create worker relationships.
If there is no fitting plan, use the existing evidenced contract workflow in
[vendor contracts](vendor-contracts.md). Do not wait for a catalog entry or create a guessed
provider tariff. An unsupported economic case is a precise gap, never permission to write journals.
On one document that mixes fees with employment cost, record the whole invoice once with
`payroll.record_statement` under its source ID, with the fee as a `fee` line; do not also bill it
through the fee plan. Use the plan's contract only for fees the provider invoices separately.

Provider evidence starting points:

| Product | Fetch and reconcile | Keep distinct |
|---|---|---|
| Deel EOR / contractor management | [Invoices](https://developer.deel.com/api/reference/endpoints/accounting/get-invoice-by-id-v-2026-01-01), contract IDs and refund statements | Deposits, funding, actual employment cost, management fee and provider FX; invoice-to-worker detail can depend on API version |
| Remote EOR / contractor management | [EOR breakdown](https://developer.remote.com/docs/pull-eor-cost-breakdown), billing documents, contractor invoices | Estimated prefunding versus actual reconciliation; breakdown can omit fees, deposits or VAT, so reconcile to the original invoice |
| Rippling US PEO | [Workers](https://developer.rippling.com/documentation/rest-api/reference/list-workers), available payroll registers, PEO invoice and bank evidence | User identity versus worker engagement; PEO and ordinary payroll are different products; obtain missing financial exports rather than assume a public endpoint |
| Justworks US PEO | [Invoice overview](https://help.justworks.com/hc/en-us/articles/360004479832-Invoice-Overview-for-Justworks-PEO), payroll reports and actual debit evidence | Net pay, employee withholding, employer burden, benefits and administrative fees; partner API access is not universal |
| Gusto US payroll | [Payroll API](https://docs.gusto.com/embedded-payroll/reference/get-v1-companies-company_id-payrolls), processed payroll/register and funding evidence | Gross cost versus net wages and deductions; ordinary payroll coverage does not imply Gusto PEO access |

For any other provider, establish the same facts from its own tools and documents. Say which
facts were obtained, which scopes/exports are missing and which exact command cannot represent
a remaining case. Do not infer legal employment or a creditor from the provider's brand.

## 2. Keep one person across arrangements

Find the existing party before creating one. Founder, employee, contractor and ownership-holder
roles can coexist. Ask the founder only when team inclusion is undecided: an invoicing founder
or embedded bookkeeper can belong to People; an external accounting firm need not.

- `people.set_membership` records the business's dated team inclusion. `through` is exclusive;
  ending an arrangement does not by itself remove a person from the team.
- `people.record_arrangement` records the dated employee, contractor, PEO or EOR relationship.
  Preserve `personPartyId`, `providerPartyId`, `employerPartyId`, `creditorPartyId` and the
  existing `contractId` where evidenced. A separate founder-owned consulting company remains
  the legal creditor, linked to the person doing the work.
- A provider move changes arrangements, not the person's party. Ending/revising a membership
  uses its live `eventId` as `expectedEventId`; retain the source and evidence of the change.
- Read `people` with `list`, `memberships` or `arrangements` and `as_of`. Active, former and
  all rosters are bounded pages. Team inclusion grants no access to the books.

## 3. Record costs once

Choose the path from the facts already in the books:

1. **A new provider invoice or payroll register:** use `payroll.record_statement`. Give one
   stable source document ID, unique line keys, exact cost subtotal and evidenced cost lines.
   State the person and `month` of work when known; omit unknown attribution or period.
   Components include pay, bonus, commission, employer tax, benefits, reimbursements, provider
   fee and other costs. The business chooses expense accounts and functions. Preserve approval
   and stated performance-period evidence for variable compensation; do not calculate earnings.
2. **An ordinary contractor/founder bill or service fee already recorded:** retain its normal
   contract/bill/recognition/payment records. Find each expense posting through `ledgers`
   `statement`, then use `payroll.link_recorded_cost` with its `postingId` and, for a translated expense, `nativeCost` from the original invoice. The posting and its original frozen conversion evidence remain intact. It adds People
   attribution without any expense or debt. Link a later ordinary contract credit's expense
   credit posting separately. Do not also record the same charge as a provider statement.
3. **Later detail for a provider aggregate:** use `payroll.attribute_cost` on its unattributed or known-person/unknown-period
   statement/line. Allocations must equal the existing cost exactly; include any unresolved
   remainder without a person. An omitted `month` keeps the line's known month. On a partly
   credited line, allocate what remains uncredited. Keep the returned `lines[].lineKey`: later
   credits and attributions name it. This adds no financial posting. Do not create a second bill.

For an EOR invoice actually owed in full to the provider, the default creditor is that provider
on trade payables. For a US payroll register, explicitly supply the evidenced liabilities:
net wages on 2170, employee withholding on 2171, employer liabilities on 2172, and provider fees
on 2110 where appropriate. The actual creditor of each component comes from evidence. These
liabilities must equal gross cost; employee deductions are part of gross pay, not extra expense.
Never record both a full provider payable and the same underlying payroll debts. Give the
evidenced `dueDate` when known; aging can only say a claim is overdue from a stated date.

Known totals remain visible even when worker details are missing. Record the unexplained cost
without a person and mark the founder's answer incomplete. Never divide it by headcount.
Keep the work month separate from invoice, payroll and payment dates. A missing work period
stays unknown. Use an evidenced `exchangeRate` when required, or the shared rate store; retain
native invoice money and the frozen booked functional amount. When worker pay is in another
currency but the provider bills the company in its own currency, preserve the original amount,
currency and provider rate evidence in the line's `providerConversion`. This is separate from
invoice-to-functional FX and does not create a second debt to the worker. A missing required rate is a gap to resolve,
not a guessed conversion.

## 4. Funding, application, returns and corrections

Follow [payments](payments.md) for actual money movements. A funding request or estimate alone
is neither payroll expense nor evidence of money moving.

- Record an actual advance/deposit with `payments.record`, outgoing and initially unallocated.
  `payroll.link_funding` labels its original receivable `prefunding` or `deposit`.
- Match actual billed cost with `payments.allocate` against the existing payable claims.
  Allocation moves no cash and recognizes no second expense. Apply a deposit only if the
  provider's evidence authorizes that application; its label alone does not authorize use.
  Funds held by a provider can settle the 2170-2172 claims of its own statements, and its incoming
  money can settle the refunds its credits open; the claims keep their actual creditors.
- Record a return with an incoming `payments.record` allocated to the original receivable.
  Returning unused funds is not income.
- Use `payroll.reverse_statement` for an evidenced full or partial provider credit. It cancels unpaid
  debt and opens refund receivables for paid amounts, preserving the payments and negative
  original-period costs. Record an evidenced replacement under its own source ID if needed.
  Once a refund is open, do not reverse the payment: settle or correct the refund claim.
  For a partial credit, give explicit `costs` line amounts and `liabilities` original claim
  amounts with equal totals. Omit both to credit all remaining amounts. Successive credits
  retain original evidence and cannot over-credit. Do not invent a replacement invoice.
- Retain the same source ID for a retry. A changed command key does not make a second invoice.
  A conflicting source is a refusal to investigate, not a reason to invent a different ID.

## 5. Verify and connect vendor usage

Read `people_costs` with a bounded `from_month` / `through_month` window and follow `nextCursor`.
Show booked functional cost, native amounts, components, unattributed amounts and unknown work
periods. Unassigned work-period costs remain visible even if the bill is dated outside the
requested month; never make that monthly answer appear complete by hiding them. `payroll_provider` pages the provider's claims, labelled deposit/prefunding, amounts
owed and recoverable refunds; its first page carries complete totals by currency and purpose,
for a party with a recorded statement, funding or arrangement. Do not
present an individual claim page as its grand total. Roster access alone
uses the parties read permission; costs and provider claims separately require their financial reports.
Reconcile source costs, claims and independent cash, and run the standard [verification](verify.md).

For named SaaS seats, read the service's own connected tools and retain its account-scoped user
ID. Use `people.link_identity` with vendor, account, holder and assignment dates; choose person,
external, shared or agent from evidence. Read `people` `identities` at the seat-review date: the
response distinguishes active, former and non-team people. A wrong mapping is corrected with
`through` equal to `from` and its `expectedEventId`, then a new record. Never rewrite old usage `holder`
values after learning a match. Keep unresolved identities unresolved. Preserve billed seats,
assigned seats and observed use as separate facts; headcount is a comparison, not a billing meter.

Continue the existing [vendor usage workflow](vendor-usage.md).
A former person with a paid seat is a review finding; a legitimate external collaborator or shared
agent can still need access. Report proposed savings without cancelling services or revoking
access. Person/holder links enable subsequent combined tool and employment cost analysis; do not
repost tool expense into payroll or force shared costs onto one person.
