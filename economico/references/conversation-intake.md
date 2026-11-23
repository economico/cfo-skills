# Conversation intake

Use the provider-neutral conversation tools only when server discovery advertises them. They
retain received and sent observations; they do not supply mailbox credentials or a running
provider connector. Keep the ordinary inbox unchanged and gather mail with the founder's
separately authorized agent. Rehearse on a disposable business, never production.

1. Discover `context_readers` and `context_messages` plus their command schemas. Register an
   installation with a stable provider account `mailboxKey`, not a display name. Registering is
   disabled. Enabling requires received/sent, stable IDs, headers, bodies, incremental recovery
   and scheduled capabilities; attachments may be unavailable. Reader configuration/health needs
   business read plus registered-actor/owner authority. Use the current expected revision.
2. Page `context_readers`, action `seeds`, for authorized originating sends and their exact
   references. Missing/non-unique `internetMessageId` is null, not a provider-ID substitution.
   Follow Economico-origin threads when the provider's verified headers support it. The agent
   selects other relevant conversations; selection alone never proves a record association.
3. Retain available body/raw/attachment bytes through `documents.receive`, for example as opaque
   `context_file` originals, then import the envelope with `context_messages.import`. Supply
   explicit `received|sent|unknown` direction and observed RFC threading headers. Mailbox-scoped
   provider message identity deduplicates across reader installations; subject/from/thread alone
   never matches an invoice. Keep source material separate from agent interpretation.
4. Report attachment failures explicitly and set `attachmentInventoryComplete:true` only once
   every descriptor is known. Intake enforces realistic email header limits (`from` at most 10,
   `to` and `cc` at most 50 each, address display names at most 256 B, `inReplyTo` at most 10,
   `references` at most 100, thread and message IDs at most 256 B, `subject` at most 2 KiB, and
   serialized headers within the 180 KiB response budget) and descriptor limits (`partKey` at most
   128 B, `name` at most 255 B, `mediaType` at most 127 B); over-limit inputs are rejected with a
   validation error rather than truncated (keep full raw headers in the optional raw MIME document). Imports/completion pages take at most 100 descriptors; additional
   material uses `context_messages.complete`. A duplicate-only transient unavailable observation
   does not discard retained bytes. Completion cannot regress retained material or replace it
   with conflicting bytes. Retain an available body even when attachments fail. If the first
   fetch lacked headers, completion can append `recoveredMetadata` under an active same-mailbox
   reader; preserve known identity and the original observation, then reimport real headers
   through ordinary matching. Never invent missing sender or RFC identity.
5. Later verified original-send provenance can be added by reobserving the same envelope with
   `lineage:{kind:"original_send",sendId,deliveryMessageId}` and its observed RFC identity.
   Never invent provider/RFC equivalence. Explicit matching uses `context_messages.match` with
   canonical target and attributed evidence. Preserve agent-asserted confidence; conflicts stay
   unresolved. Business-only context does not identify a customer. Do not fabricate contacts,
   names or billing roles, or amend money, agreements or reminder policy from an email.
6. Advance the agent's durable provider checkpoint only after selected imports have receipts and
   skipped messages have local outcomes. Economico progress takes only a cryptographic digest
   (`sha256:` or `blake3:` plus 64 lowercase hexadecimal digits), safe status/error codes and
   attributed coverage assertions. Never submit credentials, raw provider cursors or raw errors.
   A heartbeat does not refresh coverage or erase gaps. Explicit recovering-to-idle advancing
   coverage with `gapStart:null` asserts gap recovery, not that missing material was retained.

With explicit user authorization, use an agent-owned scheduled process that can continue after
the chat ends, not an interactive chat left open. Each run must retain authorized mailbox access
and current Economico permissions; stop on revoked access rather than bypassing it. Preserve
ordinary inbox delivery: do not redirect, move or delete messages for capture. Bound each invocation
to 100 provider messages and 20 MiB fetched bytes; artifacts retain the native 10 MiB ceiling. Keep
credentials, page state, lock and provider recovery checkpoints outside Economico. Provider API
assumptions and actual disposable round trips belong to the runner's acceptance, not a claim that
this intake API established live Gmail capture or a background service in Economico.

`context_messages`, action `get`, returns current status under the metadata document's rights;
opening body/raw/attachment metadata needs each source's own full authority. A `context_message`
reference identifies current completion/match state; the envelope's document reference identifies
exact immutable evidence. An authorized association reference remains available for correcting
business-only unresolved context. Reader references retain actor/owner plus business authority.
Follow bounded cursors, including empty continuation pages, and restart after scope/revision
changes. Reader ownership never grants all message/source bytes.

Replacement readers recover their own observed unresolved messages without changing first-import
attribution. Retry transient `busy` while bounded shared status catch-up is pending. Accepted
envelopes always return inline on `context_messages` (`get`) and `references` (`resolve`) while
the exact original document remains immutable; observation membership grants no source access.
