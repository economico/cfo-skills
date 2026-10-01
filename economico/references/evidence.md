# Evidence and source documents

Every entry the model posts should be explainable by an external artifact: an agreement, a
receipt, an invoice, a charge, a bank line. Economico stores those artifacts immutably and links
them to the commands they justify, so the books can be audited from the first day.

## Store the source

```json
{
  "name": "documents.receive",
  "idempotency_key": "doc:email:render:INV-2026-06-0042",
  "effective_at": "2026-07-01T00:00:00Z",
  "input": {
    "kind": "source_document",
    "contentBase64": "<base64 of the exact PDF or UTF-8 text>",
    "externalId": "email:render:INV-2026-06-0042",
    "title": "Render invoice INV-2026-06-0042"
  }
}
```

- `title` is what the founder will see the document listed as (one line, at most 200
  characters). Give one for an email or a scan; leave it out when the file names itself
  (a Markdown heading, a PDF with a `/Title`), and Economico reads that. Without either,
  the list shows the `externalId`.

- Accepted: nonempty UTF-8 text (plain text, Markdown, an email rendered as text) or a PDF, up to
  256 KiB. Office formats and images are refused; convert to PDF or text first.
- The bytes are stored exactly and addressed by hash. Sending identical bytes again returns
  `no_change` with the existing document id in `details`.
- `externalId` is how you find it again: `documents {action: "list", kind: "source_document",
  external_id: "email:render:INV-2026-06-0042"}`.
- Uploading a document records nothing by itself: no terms, no acceptance, no money.

For an email, store the attachment if the invoice is a PDF; otherwise store a text rendering with
the headers that identify it (From, Date, Subject, Message-ID) followed by the body.

Encode with the plainest tool: write the text to a scratch file in the workspace, then
`base64 < file.txt`. No script interpreter is needed, so a host that refuses one does not stop
the work; any other refusal is the founder's to decide. The text must be UTF-8: a refusal
saying the source is a binary office file usually means it was not, and a Latin-1 encoder
mangled a `•` or an em dash.

## Cite it

| Where | How |
|---|---|
| A contract's basis | `contracts.create` `sourceDocumentIds: ["<doc id>"]` |
| Acceptance | `evidence.acceptance: [{ "kind": "signed_doc", "ref": "<doc id>" }]` |
| Service delivered or received | `evidence.service: [{ "kind": "other", "ref": "<doc id>" }]` |
| Money moved | `evidence.payment: [{ "kind": "stripe_object", "ref": "ch_…" }]` or `{ "kind": "other", "ref": "<doc id or bank line>" }` |
| The recording's own source | the envelope's `source_document_id` on `contracts.record` |

Evidence kinds are `signed_doc`, `stripe_object`, `chain_tx`, `x402_receipt`, `tap_message` and
`other`. Use the most specific that is true; a `ref` can be a document id, a provider object id,
a transaction hash or a URL.

## Identities

| Identity | Scope | Build it from |
|---|---|---|
| `idempotency_key` | the whole business, per command | the step and the source: `model:party:cus_acme`, `email:render:INV-2026-06-0042` |
| `sourceFactId` | the whole business, per recorded fact | the source system and its id: `stripe:invoice:in_1Q…`, `render:invoice:INV-…`, `bank:2026-07-20:line-12` |
| `occurrenceKey` | one contract activity | the business identity of the happening: the invoice number, `usage:2026-08`, `schedule:<start>:<end>` |
| document `externalId` | the business's documents | the source identity: `email:<vendor>:<number>`, `stripe:checkout:<id>` |

Deterministic identities make the whole session safe to rerun: a repeat returns the original
receipt instead of posting twice.
