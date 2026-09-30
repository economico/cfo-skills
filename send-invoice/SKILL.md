---
name: send-invoice
description: Bill a customer in Economico and hand the founder the invoice to send.
disable-model-invocation: true
---

Bill the customer the founder names: invoke the `economico` skill and record the invoice on that customer's contract, as its `customer-contracts` reference describes. If the customer has no contract yet, set one up first (its `customer-contracts` reference again). Economico records the invoice and returns its statement document but never sends it: give the founder the document with the amount and due date, and send it through their own email only if they ask.
