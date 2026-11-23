# Record evidence

Record evidence keeps source material and attributed interpretation distinct. Use a disposable
sandbox business for rehearsal. Discover current schemas and permissions before writing; an
unsupported target adapter is unavailable, not permission to use a broader record kind.

1. Retain received bytes with `documents.receive`. Use `context_file` for opaque office, email
   or HTML bytes; never claim that retaining a file extracted its meaning. The artifact limit
   is 10 MiB and the shared business document budget is 1 GiB.
2. Call `context_materials.create` with `material.kind=retained` and its `documentId`, title and
   known origin. Original document identity stays unchanged; preserve the returned
   `observationId` when associating this particular capture. A URL/local locator uses kind
   `reference`; the agent opens it with its own access. Economico does not fetch the source.
3. Write an interpretation with kind `note`, clearly attributing it as an agent observation.
   Canonical record citations are separate links, not automatic authority to read their targets.
4. Use `context_materials.add_rendition` for supplied readable text. Say `partial` and provide
   `omissionReason` whenever text is incomplete; never silently shorten it and claim completeness.
5. `context_associations.add` names the canonical target URL, material ID, optional capture
   observation, relationship label and explanation. Require both native target and source rights.
   Attaching evidence changes no financial fact and grants no access. To reuse another actor's observation across targets, also supply `observationAssociationId`: an exact readable association of that same material/observation. Both witness endpoints are reauthorized. A withdrawn but readable association still works; the witness is not published in the new assertion.
6. Correct mistakes with `context_associations.correct` (old ID, full replacement and reason),
   or `.withdraw` (ID and reason). Keep the idempotency key stable across a retry; competing
   corrections conflict rather than replacing each other's work.

## Find context for a known record or party

Use the discovered `context {action: "record"}` read with the record's canonical HTTPS `target`,
or `context {action: "party"}` with a known `partyId`. There is no text query, search box, FTS or vector lookup.
Both default to 25 entries (at most 100 and 256 KiB); each call uses at most 100 eligible candidates
or grant-known positions, 16 exact index spans and 25 related targets across the whole request.
An empty page with a cursor is not the end. Keep the same target and filters, and continue only
this authorized traversal. Do not widen selectors, switch record kinds or run broad SQL to fill
an empty window. On `cursor_scope_changed`, restart under current permissions.

Party lookup reads direct context plus independently readable related agreements, invoice
representations and allocated payments. Subjects/documents retain their actual native LIST
requirements; finite known invoice-document grants use exact point lookups. Financial BOOK
classes have no standalone LIST: their membership spans require the existing native SQL
whole-books gate, every unrestricted `REPORT_SQL_REQUIREMENTS` read. Scoped or missing any
required read excludes that span before SQL; exact known targets still need full native GET.
Every association needs both live endpoint authorities, including protected source/rendition
roles. Results have no denied/hidden counts or source snippets. `scanComplete` describes this
traversal under current access, not a frozen snapshot or a business-wide total.

Each page returns a material once, with its bounded `associations` array. Preserve every
assertion's explanation and capture provenance. Across pages, deduplicate material IDs without
collapsing independent observations just because bytes match. Party assertions carry `origin`
as `direct` or `related` inside that array. Withdrawal can remove later results; new associations
or memberships require a fresh traversal.

The `context` read tool/CLI resource offers exact `material`, `association`, `observation`, `renditions` and
`text` actions. Non-note text needs an explicitly chosen authorized rendition. A rendition needs
its own document rights AND its source's full native rights. Use text cursors (at most 64 KiB per
page) and preserve completeness. Rendition pages can be empty with continuation under sparse
grants; follow continuation without treating an empty page as the end. On `cursor_scope_changed`,
restart under current permissions.

Descriptions and note previews are not proof of faithful extraction. Notes/references have a
256 KiB encoded bound including provenance; renditions have a 10 MiB encoded bound. An explicit
oversize refusal means retain less with a declared partial result, or keep an external reference.
No general keyword/full-text/vector search is provided. Do not promise operations that current discovery does not expose.


Association and observation point gets return canonical self-links and Markdown. Resolve
`context_association:[associationId]` through full live endpoint authority. The standalone
`context_observation:[observationId]` link is for its creator or business owner with current
material rights; narrow noncreators use the association link for authorized provenance.
Resolver input remains the URL only. Do not put mutation witness IDs into canonical identity.
Materials and renditions keep ordinary protected document-CID references.

Context association/observation records are linkable provenance metadata, not association targets.
Add/correction to either is refused before mutation. Notes may cite their canonical links without
inheriting destination permissions; a citation does not reveal or authorize destination contents.

## Current report links versus exact citations

Use `context_queries.create` to save a native report definition and normalized parameters.
Built-ins select `{kind:"builtin",id,revision:1}` with `operation:"reports.run"`; custom reports
pin `{kind:"custom",id,version}` with `operation:"reports.run_definition"`. Explicit dates remain
fixed. A rolling period names day/month, count 1–120 and IANA timezone; incompatible native
controls are refused rather than silently overwritten. A current link refreshes business data
under its pin, not another version's formula.

Call `context_queries` action `run` with `queryId` and `citable:true` before showing figures you
may need to cite. Preserve the returned lease ID with that exact display. Execute
`context_captures.create {leaseId}` to save its private exact bytes, then paste the returned
reference. This is an explicit command, not automatic saving on read. Exact citations retain
errors, partial/shortened outputs, source head, selected external rates and executing build.

Leases last at most 600 seconds and may disappear on restart. A new command on an expired run
returns `run_expired`; offer a distinct new run, never silently rerun and label it the old result.
Oversized citable results retain the ordinary answer with `captureUnavailable:result_too_large`;
capacity exhaustion refuses admission without evicting live leases. Keep idempotency/effective
inputs stable when retrying an accepted command, and retain native document/report authority.
A citation creates no public share. Resolve its Markdown destination with the `references` read, action `resolve`,
or read `context_captures get --captureId` in the CLI with explicit `--business`.

## Oversized metadata

A metadata page's 256-KiB budget includes escaped text, structured content and the protocol
envelope. Follow continuations; fewer items than the requested limit does not mean completion.
An oversized association reference returns status `content_required`: use `references.content`,
reassemble its raw byte pages and preserve the revision. Revoked source/target rights still
refuse the content. An association remains metadata, not an attachment target.

If a material metadata read explicitly refuses a large description or citation inventory,
retain its identity and obtain the exact original with MCP `documents {action:"get",id}`.
Decode `contentBase64` from the text result; do not substitute a shortened note or a missing
record. Original-file reads retain their native source authority and separate size budget.
