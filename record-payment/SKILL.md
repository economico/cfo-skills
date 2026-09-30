---
name: record-payment
description: Record a payment a customer made against an invoice in Economico.
disable-model-invocation: true
---

Record the customer payment the founder describes: invoke the `economico` skill and record the receipt on the customer's contract against the invoice it pays, as its `customer-contracts` reference describes, citing the bank line or Stripe charge as its `evidence` reference says.
