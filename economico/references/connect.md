# Connect and choose the business

Economico is one server with three transports over the same catalog. Pick one, verify it, and
name the business before any write.

## Which host

The new system runs at `https://ng.economi.co` until cutover; after cutover it is
`https://economi.co`. The discovery documents tell you which one you are on:
`/.well-known/oauth-protected-resource/mcp` names the resource. If the founder gives you a URL,
use theirs. Everything below uses `ng.economi.co`.

## MCP (preferred when the host speaks MCP)

```bash
claude mcp add --transport http economico https://ng.economi.co/mcp
```

The host drives OAuth: a browser opens, the founder signs in with an emailed one-time code or
Google, and approves which businesses this connection may reach and with what access
(**Read books**, **Keep books**, **Administer books and sharing**). Modeling a business needs
Keep books at least. The tools you get are `businesses`, `catalog`, `commands`,
`command_receipt`, `reports`, `export`, `ping`, and one read tool per read model (`subjects`,
`documents`, `events`, `ledgers`, `parties`, `accounts`, `categories`, `tax_codes`,
`shared_views`, `business`). Every tool except `businesses` takes a required `business` slug.

## CLI (for shell agents, CI, or several businesses at once)

```bash
npm install -g @economico/cli          # Node 24+
economico login --server https://ng.economi.co --local
economico businesses list
economico --business <slug> catalog list
economico --business <slug> catalog describe --name contracts.record
economico --business <slug> commands execute --args @command.json
economico --business <slug> reports run --kind balance_sheet --currency USD --human
```

`--local` keeps the credentials in `./.economico/config.json` (gitignored for you), so one
directory is one identity. The CLI has no command tree of its own: `--help` at any level reads the
server's OpenAPI document. `--args` takes JSON, `@file` or `-` for stdin. Command calls are never
retried automatically; retry deliberately with the same `idempotency_key`.

## REST (for a product backend)

`/v1/*` is the same OAuth resource. Name the business in `X-Economico-Business`.
`POST /v1/commands` takes the command envelope without `action`; `GET /v1/openapi.json` is the
permission-filtered description of what the token may do; `POST /v1/reports` runs a report.

## Headless agents

An unattended worker uses a confidential client: the founder registers the worker's public key
from a signed-in session, and the worker trades a short-lived `private_key_jwt` assertion for a
token at `/oauth/headless/token`. A headless token binds one business and refuses commands whose
`credential` (in `catalog describe`) is `interactive` or `browser`. Owner-only legal-identity
changes are interactive; membership, ownership and public-access changes are browser-only.

## Choosing, or creating, the business

1. Call `businesses`. It lists the businesses this connection was approved for, each with its
   slug and effective permissions. If the repository's `business-model.md` names a business
   under "For the next session" and it is listed, that is this repository's books: use it
   without asking.
2. If the founder has none, the business is created in the browser during sign-in: the form asks
   for the name, where it is incorporated, the legal form, functional currency and fiscal year
   end. The CLI can prefill it: `economico login --country US` (or `SG`, `CA`, `GB`,
   `--legal-form`, `--currency`, `--fiscal-year-end MM-DD`). A founder who has not incorporated
   picks the legal form `project`; it can incorporate in place later.
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
the grant is read-only: ask the founder to reconnect with Keep books.

## Troubleshooting

- `401 invalid_token`, "no active approved connection": the grant was revoked or replaced; sign
  in again.
- A business missing from `businesses`: it was not approved for this connection. Reconnect and
  tick it on the consent screen; approving another business never widens an existing grant.
- `plan_required`: shared views and extra members need the Team plan, which is not on sale yet.
  Nothing in this skill needs it.
- The CLI's default server is the old system. Pass `--server` explicitly.
