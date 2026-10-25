# Connect and choose the business

Economico is one server with three transports over the same catalog. Pick one, verify it, and
name the business before any write.

## Which host

Economico runs at `https://economi.co`. The discovery document
`/.well-known/oauth-protected-resource/mcp` names the resource you are on. If the founder gives
you a URL, use theirs.

## MCP (preferred when the host speaks MCP)

The Claude Code plugin (`economico@cfo-skills`) declares this server, so after installing it
there is nothing to add: sign in from `/mcp` when Claude Code lists `economico` as needing
authentication. Otherwise, add it yourself:

```bash
claude mcp add --transport http economico https://economi.co/mcp
```

The host drives OAuth: a browser opens, the founder signs in with an emailed one-time code or
Google, and approves which businesses this connection may reach. Every connection gets full
access to those businesses, up to the founder's own permissions. The tools you get are
`businesses`, `catalog`, `commands`, `command_receipt`, `reports`, `export`, `ping`, and one
read tool per read model (`subjects`, `documents`, `events`, `ledgers`, `parties`, `accounts`,
`categories`, `tax_codes`, `shared_views`, `business`). Every tool except `businesses` takes a required `business` slug.

## CLI (for shell agents, CI, or several businesses at once)

```bash
npm install -g @economico/cli          # Node 24+
economico login --server https://economi.co --local
economico businesses list
economico --business <slug> catalog list
economico --business <slug> catalog describe --name contracts.record
economico --business <slug> commands execute --args @command.json
economico --business <slug> reports run --kind balance_sheet --currency USD --human
```

`--local` keeps the credentials in `./.economico/config.json` (gitignored for you), so one
directory is one identity. The CLI has no command tree of its own: `--help` at any level reads the
server's OpenAPI document. `--args` takes JSON, `@file` or `-` for stdin. Command calls are never
retried automatically; retry deliberately with the same `idempotency_key`. The one exception is
`busy` (HTTP 503): the business was receiving other uploads or serving other large documents and
did not start the request, so the CLI waits and sends it again a few times, and so should you.

## REST (for a product backend)

`/v1/*` is the same OAuth resource. Name the business in `X-Economico-Business`.
`POST /v1/commands` takes the command envelope without `action`; `GET /v1/openapi.json` is the
permission-filtered description of what the token may do; `POST /v1/reports` runs a report.

## Headless agents

An unattended worker uses a confidential client. For an existing business, the founder
registers the worker's public key from a signed-in session. For a new business, POST JSON
under `/oauth/agent/signup/`: `start` takes `{email,jwks:{keys:[publicKey]}}`, using a distinct
`kid` and RS256 (at least 2048 bits) or ES256/P-256. Keep the private key local. `verify`
takes `{code}` from the recovery email and the returned continuation in `Authorization:
Bearer …`; replace it with the rotated continuation. `offer` takes the company setup fields
(`name`, `jurisdiction`, `legalEntityType`, `functionalCurrency`, `fiscalYearEnd` as `MM-DD`)
and returns the Solo
agreement. Review it, then `accept` its exact `{id,offerCid,capacity,authorized:true}` using
the verified continuation. It returns `businessSlug`, `clientId` and `tokenEndpoint`.
Continuations expire after 30 minutes; a wallet cannot substitute for recovery-email verification.

Trade a short-lived `private_key_jwt` assertion for a token at `/oauth/headless/token`.
A headless token binds one business and refuses commands whose `credential` (in `catalog
describe`) is `interactive` or `browser`. Owner-only legal-identity changes are interactive;
membership, ownership and public-access changes are browser-only.

To add a coworker, the owner's interactive connection can call `members_invite` with their
email and `access` (`reader`, `bookkeeper` or `administrator`; never ownership). Economico emails
the address; the coworker signs in or creates an account with it and accepts in the browser, and
nobody has access until then. `member_invitations` lists pending invitations and
`members_revoke_invitation` withdraws one. Members beyond the owner need Startup: on Solo the
tool answers `plan_required` with a pricing link, which is information, not a purchase.

## Buying a plan or paying an invoice

Use `platform_purchase` with the selected `business`. A live owner-full grant can request an
`offer` (`plan: "team"` for Startup, `"events"` for Solo extra events), read its terms, `accept`
the exact `offerId` and `offerCid` with `capacity` and `authorized: true`, then `prepare` and
poll `status` using that offer. Every business starts Solo; new Startup offers have no ordinary
trial. Server-verified YC eligibility determines any concession.

Use `pay` with `method: "card"` for a saved card, or follow `browserFallback` for card setup
and bank confirmation. Use the returned `paymentUrl` for an advertised
wallet method. Wallet credentials belong in that separate authenticated HTTP exchange:
`Payment` for MPP or `PAYMENT-SIGNATURE` for x402, preserving the OAuth bearer. Let the wallet
apply its approval policy. `methods` describes configured options; public access to Link/MPP
and x402 depends on merchant enablement. Use `status` and its paged `invoices` for later usage
and renewal payments. An unknown payment outcome must reconcile before changing methods.
Startup activates after the first invoice is paid; a trusted zero-due YC offer needs no
payment. Retry blocked work after activation. A temporary activation failure retries service
without paying again; a return obligation needs support recovery.

## Choosing, or creating, the business

1. Call `businesses`. It lists the businesses this connection was approved for, each with its
   slug and effective permissions. If the repository's `business-model.md` names a business
   under "For the next session" and it is listed, that is this repository's books: use it
   without asking.
2. If the founder has none, the business is created in the browser during sign-in: the form asks
   for the name, where it is incorporated, the legal form, functional currency and fiscal year
   end. The CLI can prefill it: `economico login --country US` (or `SG`, `CA`, `GB`,
   `--legal-form`, `--currency`, `--fiscal-year-end MM-DD`). A founder who has not incorporated
   picks the legal form `project`; it can incorporate in place later. Get the functional currency
   right here: it can be corrected only while the book is empty, and a move to another economy
   later is a new business.
3. To rehearse, create a second business named for what it is ("Acme sandbox") and approve it
   for this connection. There is no scenario or sandbox mode inside a business: a rehearsal is a
   separate business, and it costs nothing.

State the target out loud before the first write: "Writing to `acme-sandbox-3f2a` (sandbox), not
`acme-inc-91bd`."

## Verify the connection

```json
{ "tool": "business", "business": "<slug>", "action": "get" }
{ "tool": "catalog",  "business": "<slug>", "action": "permissions" }
{ "tool": "reports",  "business": "<slug>", "action": "list" }
```

`business get` returns the legal form, `functionalCurrency`, `primaryLedgerId` (the GAAP ledger
contracts bind; USD is `led_00000000000000000000000840`) and `fiscalYearEnd`. `catalog
permissions` shows exactly what this token may do. If `commands` is missing from the tool list,
the grant is read-only: a member whose own permissions are read-only, or — on a Startup-plan business —
an owner who chose the Read books preset on the consent screen instead of Owner. If `business
get` itself is refused but `reports list` works, the grant is Reports only — the narrowest
preset, with no access to raw book data at all. Ask the founder to reconnect with a wider
preset, or to widen the membership.

## Troubleshooting

- `401 invalid_token`, "no active approved connection": the grant was revoked or replaced; sign
  in again.
- A business missing from `businesses`: it was not approved for this connection. Reconnect and
  tick it on the consent screen; approving another business never widens an existing grant.
- `demo: true` on the `business` read: a demo company with made-up figures, the user's alone.
  Say it is a demo whenever you report its numbers (its books run up to `demoOpenedAt`).
  `demo_business` means a demo may not do that (reach another business, send a message other
  than a support request or feature suggestion to Economico, add a member, publish, take a
  plan, or look like a real company) or has recorded the 2,000 events a demo may record; tell
  the user they can create their own business for it. Economico cannot reply to a demo's
  message, so tell the user to write from their own business when they need an answer. A demo
  delivers a few such messages a day; past that its message is `undeliverable` with the reason.
- `plan_required` has two causes. With `details.through`, the business has used its monthly
  event allowance: nothing was recorded. `details.reason` says why. `solo_limit`: Solo's
  allowance; tell the founder the limit (`details.included` events, `details.used` already used)
  and the reset date (`through`), then offer the returned `details.choices`: wait until `through`,
  or use `platform_purchase` to upgrade to Startup or accept paid extra events. `plan_cancelled`: Startup was cancelled, or Solo's extra events were
  turned off; the extra events so far were billed then, and until the plan ends
  (`details.endsAt`) only `details.allowed` events in each billing period are allowed (the
  included events, or more in the period the cancel billed a started block); retry after
  `through` (the next period's included events) or after `endsAt`, when the business is on Solo
  (`details.after: "solo"` for Startup). Before a batch, `business {action: "meter"}` shows the
  same ceiling as the period's `allowed` while the cap holds. Tell the founder; the billing page
  says whether the plan can be kept: once the extra events billed then are paid (for monthly Startup whose cancellation billed
  extra events, keeping it takes a new Startup agreement from the end; a first-year YC price cannot
  carry over, so that plan ends as scheduled), and a kept plan stays at its included events until
  the settled period ends. Any new paid plan (Startup, extra events, a switch or that new
  agreement) starts only once every invoice from Economico is paid; the billing page says so.
  `usage_settled`: Startup or Solo's extra events were kept after a cancel; retry after `through`.
  With `details.closesAt`, the business is set to close then (its owner approved closing it): tell
  the founder, and do not wait for `endsAt` or suggest keeping or upgrading the plan; the owner can
  stop the closing at `/account/cancel` until then. Reads and exports keep working meanwhile, and
  upgrading too unless the business is set to close. Without
  `details.through`, the command is a Startup feature (shared views and extra members); nothing
  else in this skill needs it.
- `validation` with `details.reason: "economico_delivers_plan_invoices"`: you tried to record a
  bill on Economico's own plan contract. Nothing was recorded. Economico's invoices arrive in the
  books by themselves; never record or import them. Pay one that is owed with `payments.record`.
