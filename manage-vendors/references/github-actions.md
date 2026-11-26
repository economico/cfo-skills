# GitHub Actions: amount or meter for one charge

The `team` template includes `actions-amount` for an evidenced net charge and `actions-linux`
for priced Linux minutes. Agent decision: choose one representation for each evidenced charge,
including when detailed usage arrives after an amount was recorded. Before **either** recording,
read the vendor's existing contracts and occurrences for that account and billing month,
including historical standalone Actions contracts. Read `usage_estimates` for each matching
contract with `contract_id`, `from` and `until`, paging its rows and `unbilledEstimates`;
`activity_claims` alone misses observed-only usage. Match invoice/report identities, periods,
SKUs and any amount already recorded. Different rights, source IDs or contract IDs do not prove
separate charges. The kernel raises `invoice_overlap` when an amount bill and observed usage share a charge
source ID, or share invoice/report evidence and matching period totals with the bill contained
in one usage period
on the same expense account, vendor and monetary ledger. It checks both recording orders and
separate contracts. Different source IDs without shared evidence can still describe one charge;
check those and statements spanning several usage periods manually. Shared-report evidence
needs a retained nonzero cost contribution on that monetary ledger; a free measurement alone
does not establish duplication. `contracts.observe_usage` alone
posts the priced expense and accrual before any invoice or claim exists: a recorded 600-cent
amount followed by a 600-cent observation produces 1,200 cents of expense, in either order.

If `actions-amount` already covers the charge, retain the later usage report with
`documents.receive`; do not send those minutes to `contracts.observe_usage` or record them as
another economic usage, bill or recognition. Link the report using `contracts.reconcile_usage`:
mark the already recorded charge's line `handledElsewhere: true`, give `invoiceTotal`, set
`source.documentId` to the retained document, and include it as
`{ kind: "other", ref: <documentId> }` in `evidence.billing`.
If the meter already covers it, reconcile its first invoice with `contracts.reconcile_usage`
against the observed period and quantity, which clears the accrual rather than adding the
expense again. If that period is already billed, retain a duplicate receipt with a
`handledElsewhere` line; record an evidenced difference as a signed delta on a metered line
with `correction: true`. A +12-cent correction to the billed 600-cent period makes expense
612 cents. Never bill the amount right as well. For unresolved overlap, retain the documents
and raise it for the
business's attention before adding a second expense. Do not silently reverse a valid bill,
invent an allocation or infer minutes by dividing money by a rate. Distinct, evidenced charges
may use separate rights, including in one month; a shared month alone is not a refusal rule.

The amount is already net of included compute and consumes no compute-credit lot. Keep its
billing, usage and recognition on the same obligation with the same evidenced period and amount;
settle the existing claim. This procedure is agent guidance, not a kernel duplicate guard.
