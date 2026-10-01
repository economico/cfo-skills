# Explore email

The founder's inbox holds the company's third-party record: receipts and invoices from vendors,
welcome emails that name the plan they signed up for, signed agreements with customers, bank and
card notices. You read it with the founder's own access (a Gmail or Outlook MCP server, or an
exported folder of `.eml` files). Economico has no inbox connection and keeps no standing
credentials; each receipt you record carries its own source document.

Read only. Never send, archive, label or delete mail unless the founder asks.

## Searches that find the money

Start broad, then narrow by vendor. In Gmail syntax, the first three find money, the fourth
finds plans and terms, and the fifth finds signed agreements; paste each line as it is:

```text
subject:(receipt OR invoice OR "payment received" OR "your bill" OR "order confirmation") newer_than:1y
from:(billing OR invoice OR invoices OR receipts OR no-reply OR noreply OR payments) newer_than:1y
"amount paid" OR "total due" OR "amount due" OR "charged to" newer_than:1y
subject:(welcome OR "you're subscribed" OR "subscription confirmed" OR "trial")
subject:(signed OR "completed:" OR docusign OR "pandadoc" OR "order form")
```

Then one search per vendor found in the code (`from:(@render.com)`, `from:(@anthropic.com)`) to
collect the full series of that vendor's receipts.

## What to extract from each message

| Field | Why |
|---|---|
| `Message-ID` header | The unique source identity: `sourceFactId` `email:<vendor>:<invoice or message id>` and the document's `externalId` |
| Vendor name and sending domain | The party (`ven_<domain slug>`, `iri` `https://<domain>`) |
| Invoice or receipt number | The occurrence key, and the first dedup key |
| Issue date, service period, due date | `effectiveAt`, the period, and `due` for bills on terms |
| Currency and total; line items if several | The amount facts; separate lines that map to different accounts |
| "Paid", "charged to Visa ending 4242", "auto-debit", or "amount due" | How it was paid: see below |
| Tax lines | Recoverable input tax vs part of the cost: see [accounts](accounts.md) |
| Links to terms of service and pricing | The vendor contract's source documents |

Use the attached PDF when there is one; otherwise the message body as text. Keep the raw bytes
you will store under 256 KiB.

## How it was paid decides the recipe

| The message says | It is | Recipe |
|---|---|---|
| "We charged your Visa/Mastercard ending 4242" and 4242 is a company card | Card expense, nothing owed to the vendor | [vendor-receipt-company-card](recipes/vendor-receipt-company-card.json) |
| "Paid", "auto-debit", "ACH debit" from the company bank | Expense and payment in one occurrence | [vendor-receipt-paid-from-bank](recipes/vendor-receipt-paid-from-bank.json) |
| "Amount due", "due by", "net 30" | A bill on terms; pay later | [vendor-bill-paid-from-bank](recipes/vendor-bill-paid-from-bank.json) |
| Charged to the founder's personal card | The vendor's bill, then the founder's payment of it; owed to the founder until repaid or contributed | [founder-paid-expense](recipes/founder-paid-expense.json) |

If you cannot tell which card was charged, ask once which cards (last four digits) are the
company's and which are the founder's own, and apply the answer to every receipt. Register the
founder's with `ownerPartyId` ([vendor contracts](vendor-contracts.md)).

## Dedup before you record

1. Look for the source: `documents {action: "list", kind: "source_document", external_id}`. If
   the exact bytes were already received, `documents.receive` returns `no_change` with the
   existing document id.
2. Look for the occurrence: `subjects {action: "list", type: "activity_occurrence", contract_id}`
   on the vendor's contract, and match the invoice number (your occurrence key).
3. With no invoice number, match amount plus service period plus vendor. A monthly receipt with
   the same amount as last month is a new occurrence, not a duplicate: the period differs.
4. Forwarded copies, "receipt" plus "invoice" emails for the same charge, and the card
   statement line are one economic event. Record it once and cite the others as evidence.

## Customer agreements in email

A signed order form or MSA (DocuSign, PandaDoc, Common Paper, a countersigned PDF) is the
acceptance evidence for a customer contract. Store it with `documents.receive` and cite it in the
contract's `sourceDocumentIds` and in the acceptance recording. Its terms override the list price.
