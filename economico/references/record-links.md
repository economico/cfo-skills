# Resolve record links and page content

Pass a Markdown link's canonical HTTPS destination to the discovered `references`
tool with `action: "resolve"` and explicit `business` (`references_resolve` with individual tools).
In the CLI:

```sh
economico --business <slug> references resolve --url '<destination>'
```

Current stored-record families have typed native or basic views: business/subjects, parties,
accounts/wallets/ledgers/postings, documents/history/ingestions, claims/payments/card statements,
People/payroll, credit/usage, settings/reporting, access records and customer mail. Saved `query`
links run their saved registered operation and parameters over current data (report queries also
pin their definition); `capture` links read retained exact results without recomputation. Use only operations advertised by current discovery. These reads
do not create material, save query/capture results or perform general keyword/full-text search. Use [record evidence](record-evidence.md) for context workflows.
Use the returned canonical URL and Markdown rather than guessing a kind/key from a similar ID.

The URL names the same business you explicitly select and grants no access. It contains no bearer
credential and never creates public access. Native read permissions, stored selectors and
human-only restrictions still apply. A signed-in person gets a readable basic page with recorded
fields, identity, source/related links and copy controls. Do not scrape that page or fetch another
host as a fallback for a refusal. Read private titles only after a successful authorized resolve.
Document metadata GET needs no LIST authority; content bytes remain an explicit bounded read.

## Read large scalar content

For new scalar projections, a resolve response exceeding **256 KiB** returns
`status: "content_required"`, `value.data: null` and
`value.content: {url,mediaType:"application/json",byteLength,revision}`. Identity, provenance
and authorized links remain present. The absolute content URL is
`https://economi.co/v1/references/content?url=<canonical>` (query-encoded). The 25 native-backed
payloads keep their own limits unchanged.

Use `references` with `action: "content"` (`references_content` with individual tools):

```json
{"business":"acme","action":"content","url":"<canonical destination>","byte_limit":65536}
```

REST is `GET /v1/references/content`; the discovered CLI is
`economico --business <slug> references content --url '<destination>' --byte_limit 65536`.
It returns `{reference,mediaType:"application/json",revision,contentBase64,nextCursor}`.
The raw-byte limit is **1–65536**, default **65536**, distinct from the children's item-count
`limit`. Follow opaque cursors until null. Base64-decode each page and concatenate its bytes
before UTF-8 decoding and JSON parsing: a page can split a character or token.

The content is exact bigint-normalized JSON of the whole typed value
`{schemaVersion,record,data,provenance,links}`, not raw storage or arbitrary fields.
Cursors bind reference, revision and byte offset. After `cursor_stale`, explicitly restart
without the cursor; never mix revisions. Every page rechecks full live record authority,
including after revocation. Neither the descriptor nor cursor grants access.
The browser has an explicit bounded structured-content read and Next pages while retaining
readable metadata, identity and links in its main view, not raw JSON alone.

## Page a recorded collection

Use `references` with `action: "children"` (`references_children` with individual tools):

```json
{"business":"acme","action":"children","url":"<canonical destination>","child":"positions","limit":25}
```

Or `economico --business <slug> references children --url '<destination>' --child positions`.
The result is `{reference,child,items,nextCursor}`. Closed names belong to these parent kinds:

| Parent | Child |
|---|---|
| party | `contacts` |
| claim | `positions` |
| payment, payment_allocation, payment_unallocation, payroll_attribution | `allocations` |
| payroll_statement | `costs` |
| payroll_correction | `costs`, `liabilities` |
| period_fact | `facts` |
| usage_reconciliation | `sourceLines` |
| customer_mail_send | `contactIds` |

Each page uses the parent's full GET policy, without a new LIST grant or recursive target
expansion. Default limit is **25**, maximum **100**. The serialized response has a **256 KiB**
ceiling, so a page can be shorter than its count limit. Continue with the opaque `nextCursor`
(`--cursor '<nextCursor>'` in the CLI) until null; do not invent a total from the first page.

Cursors bind the business, full parent identity, child, revision and seek position. After
`cursor_stale`, explicitly restart without the old cursor; never combine pages from different
revisions. The browser offers Load more and Restart collection and warns about retained stale
rows. Incomplete historical projection catch-up is not evidence of an empty collection. Do not
truncate oversize fields.

Children can also return `contentLinks:[{record,url}]` (at most one per page) when the first
row is oversized. `record` identifies the standalone `party_contact`; `url` uses the existing
`/v1/references/content` endpoint with that child's canonical URL. A page before an oversized
row returns preceding small items and a cursor before it. An oversized first row returns
`items: []`, an actionable link and a cursor past the row, keeping following rows reachable.
There is no truncated field or partial JSON item. Following the child rechecks its full live
authority; native party GET is unchanged. The browser links to the child's readable record
view and explicit structured-content Read, preserving continuation and the deferred-row count.
This is the existing content read, not a child-content mode or authorization bypass.

Claim positions expose recorded `original`, not current outstanding. A claim may have multiple
currencies or components: do not invent one party/currency or sum mixed units. Payments expose
recorded amounts and bounded allocations, not embedded evidence or computed current balances.
Use the existing native reports/ledger reads for current calculated figures.

## Exact identities and discovery rights

A contact is `[partyId,contactId]`; a claim position is
`[claimId,originLedgerId,accountCode]`; a posting is `[eventCid,ordinal]`. Keep numeric codes and
ordinals as exact strings. Vendor invoice lines use `[sourceEventCid,lineKey,lineOrdinal]`, not
reservation IDs; when the source has no amount the view says `sourceShape: "reference"`, and you
must not manufacture an amount.

A document relation is `[fromDocumentId,toDocumentId,relationKind]`, **not** its creating event
CID. Both endpoint native document-read policies must pass, including their stored document-kind
restrictions. Reading one endpoint does not authorize the other. Standalone document links use
document CIDs, not native document IDs. Provenance and related links require their own authorized
reads; identifiers do not grant their targets' titles, source payloads or bytes.

Known-target GET is **not LIST discovery**. For contracts/documents, pass their actual native
LIST operation's requirements, then full returned-target GET for each result. Claims and payments
have no native LIST; discovery must independently pass unrestricted whole-books SQL enumeration
(`REPORT_SQL_REQUIREMENTS`): unrestricted reads of documents, events, subjects, ledgers, parties,
accounts, payment routes, categories, tax codes, business, receipts and export. Then check every
returned-target GET. Caller-supplied exact IDs can be
read as known targets, but do not grant enumeration authority. Never replace a narrower native
GET policy with BOOK, or treat a successful GET as permission to discover related records.
