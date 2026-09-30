---
name: record-receipts
description: Find vendor receipts and bills in the founder's email and record them in Economico.
disable-model-invocation: true
---

Record the receipts and bills in the founder's mailbox for the period they name (last month if they name none): invoke the `economico` skill, find them as its `explore-email` reference describes, and record each on its vendor's contract per its `vendor-contracts` reference, skipping any already recorded.
