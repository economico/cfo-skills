# Modeling products and pricing

A product plan is a reusable agreement template containing the rights a customer receives.
Model the rights before recording invoices or payments. A monthly platform access right can be
billed in advance, earned over the month and paid later without becoming three activities.

## Identify rights and prices

List what the party receives, the evidence, price treatment, billing timing and recognition
basis. Separate access from measured overage when they are distinct rights. Keep included,
free and promotional rights visible. Preserve the agreement's explicit treatment: `included`
means supplied as part of that agreement, `free` means independently granted without charge,
and `promotional` means an evidenced promotion. Zero money alone does not make these interchangeable. Mark incomplete pricing as unknown rather than inventing
a rate or dropping the right. Two real rights may use identical posting shapes; do not merge
them because their account codes match.

Use `model: "priced-rights-v1"` on activity and template definitions. An activity's `price`
contains `treatment`, and for a priced right, a fixed/rate/tiered calculation and currency term.
Agreed values live in `terms`; measured quantities live in `facts`. State quantity scaling in the
right's description: for example, the consulting recipe uses integer millionths of an hour,
so 1,000,000 measured units and `unitsPerPrice: 1000000` mean one hour at the stated rate.
Keep the same scale in each phase and verify a sample quantity against the source price. Classification and payment
method are not rights. Neither is a currency variant, invoice, acceptance or cancellation.

- Fixed access: one service right with its agreed price, independent billing and recognition.
- Usage: a service right with measured quantity and an evidenced rate/tier schedule. Record
  usage, recognition and billing for the same obligation/period at their respective instants.
- Per-seat access: price from evidenced seat quantity and rate. Use evidenced prospective
  amendments at service/billing boundaries, preserving earlier terms and recorded obligations.
  Consequence-bearing rights require explicit adjustment.
- Annual prepayment: annual billing plus ratable recognition under the agreed service schedule.
  Do not manufacture twelve posting activities or recognize the whole year when paid.
- A period minimum is a bounded consequence of the period's rights, not another invented service.
- Included/free rights retain their identity and explicit treatment even when priced at zero.

For an evidenced included/free/promotional quantity, bind a capacity right to its native
capacity ledger. `grant_capacity` specifies `capacityLedger`, `quantity` and `expiresAtTerm`;
`consume_capacity` specifies that ledger, quantity and a typed `grantOccurrenceFact`. Record
these as exercise consequences with service evidence and explicit consequence selection.
`expire_capacity` is a period-close consequence with economic evidence. Check `activity_capacity`
for each lot's free, consumed and expired quantities. For prepaid credits, bill once, settle independently, and grant using that billed obligation
key. A priced capacity right declares usage recognition and usage revenue classification.
Each measured consumption releases its share of the original invoice value, with exact final
rounding; it never invoices or collects again. Expiry alone grants no breakage or refund
authority. For included usage, declare `overageRight` on consumption and use `capacity_excess`
pricing on that separate usage right. Record the full measured quantity once; the kernel
freezes covered and excess units. Later usage, recognition and billing reference that consumption
through its typed `consumptionFact` and use its occurrence ID as `obligationKey`. Never invent
the excess quantity or record a second allowance draw just to bill it. Expired lots cover zero.
Not every historical allowance/capacity workflow has a complete priced-right replacement yet.
If the live catalog cannot execute a required economic consequence, retain the source and gap;
do not imitate it with arbitrary postings or suppress the right to make a recipe pass.

## Reuse and identity

A template follows an agreement boundary. Reuse across customers when their rights and timing
are the same. A negotiated price belongs in per-right term overrides. Split an agreement only
when evidence shows distinct agreements or materially different rights, not for different bank
accounts, tax classifications or GL accounts. Preserve several agreements with one party where
that is what the source shows.

Name keys for the rights (`access`, `api-use`, `hosting`), not for accounting stages (`bill`,
`pay`, `accept`). A different spelling is not proof of a different right: explain its economic
meaning from source evidence. Keep external product/price identifiers as provenance where the
catalog supports them.

## Build and prove

1. Describe the live activity, template and contract commands. Create immutable rights, bind
   them in the template, and create a real party's contract at its agreement start date.
2. Accept through `contracts.accept`, with the current contract document ID and evidence.
3. Record delivery/usage, billing and recognition against consistent obligation/period identity.
   Give each phase a stable distinct source identity. Recognition never creates a second invoice.
4. Read claims and use independent [payments](payments.md). Test a different supported paying
   account without changing the contract. Check exact balances and outstanding claim amounts.
5. Compare the recorded economics with the source: what was received, owed, earned and paid,
   including missing evidence. Do not declare success from a template's name or an empty model.

The [vendor service examples](vendor-contracts.md) and the validated recipes below are
executable priced-right command sequences. Recipes still marked historical are not instructions
for new agreements. A recipe's `steps` run in order; `{"$ref":"step.field"}` copies an actual
prior result, including array paths such as `claim-read.claims.0.component`. Report `expect`
values assert exact books, not just successful commands.

## Recipes

Validated service examples: [monthly subscription](recipes/customer-monthly-subscription.json),
[annual subscription](recipes/customer-annual-prepaid.json),
[consulting hours and milestone](recipes/customer-consulting-hourly-milestone.json),
[seat expansion](recipes/customer-per-seat-subscription.json),
[platform and usage](recipes/customer-platform-fee-plus-usage.json),
[prepaid credits](recipes/customer-prepaid-credits.json),
[included usage and overage](recipes/customer-included-usage-overage.json),
[Stripe gross charges, fees and payout](recipes/stripe-gross-net-payout.json), and
[annual vendor prepayment](recipes/vendor-annual-prepay.json).
