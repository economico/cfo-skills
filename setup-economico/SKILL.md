---
name: setup-economico
description: >
  Connect and authenticate Economico — the agent-native general ledger that the cfo-skills
  (pricing, expense tracking, invoicing & billing, forecasting, reporting) operate against.
  Use when setting up or connecting Economico for the first time, wiring up the books,
  authenticating an agent, or verifying the connection before doing CFO work. Triggers on:
  "set up economico", "connect economico", "claude mcp add economico", "economico login",
  "install the economico cli", "@economico/cli", "https://economi.co/mcp", "economico oauth",
  "authenticate the ledger", "headless / unattended economico agent", "confidential client",
  "private_key_jwt", "service account for economico", "verify economico connection",
  "which economico account am I on". Pick this skill before any other cfo-skill when the
  ledger isn't connected yet — the money-loop skills assume an authenticated Economico.
---

# Setup Economico

Economico is an agent-native general ledger. Before any other cfo-skill can invoice a
customer, book a bill, or pull a report, the agent must be **connected and authenticated**
to an Economico business. This skill gets you there and verifies it worked.

There are two mutually-exclusive connection paths. Pick **one**.

| Path | When to use | What you talk to |
|---|---|---|
| **MCP** (preferred for this agent) | MCP-capable host: Claude Code, Claude Desktop, Cursor, ChatGPT, Lindy, OpenAI Agents. | `https://economi.co/mcp` over HTTP with bearer auth |
| **CLI** | Shell / CI runner / coding agent with shell access but no MCP — **and the best path when you run more than one business** (see below). | `@economico/cli` npm package — `economico <command>` |

Both expose the same surface — the same money-loop verbs and JSON shapes — and both
authenticate via OAuth 2.1. Default to **MCP** when this host supports it; fall back to the
CLI only when it doesn't.

### Tradeoff: multiple businesses → use the CLI with per-project credentials

One person often runs several companies. The two paths isolate them differently:

- **MCP** stores its token keyed by the **server URL**. Since every business lives behind the
  same `https://economi.co/mcp`, an MCP host keeps **one** Economico identity at a time —
  re-authenticating for company B overwrites company A. Good for a single business, awkward
  for many.
- **CLI** can scope credentials **per directory**. Run `economico login --local` from a
  project folder and the tokens land in a gitignored `./.economico/config.json` there instead
  of the global `~/.config` file. Every command run from that folder — or any subfolder —
  auto-discovers the local config (walking up the tree like `git`), so each project stays
  signed in to **its own** business with no flags to remember.

So: **one business → either path; several businesses you switch between by directory → the
CLI is the better fit.** See "Per-project credentials" under Path B.

Override the deployment with `ECONOMICO_API_URL` (CLI) or by swapping the `/mcp` host —
default is `https://economi.co`.

---

## Path A — MCP (preferred)

The host runs the full OAuth dance itself once you give it the MCP URL. There are **no MCP
tools for sign-up or sign-in** — the host hits `/mcp`, gets `401` + a `WWW-Authenticate`
pointing at the OAuth Authorization Server, runs RFC 7591 Dynamic Client Registration,
opens `/oauth/authorize` in the user's browser; the user enters their email + the 6-digit
OTP and clicks **Allow**; the host receives an access + refresh token pair.

```bash
claude mcp add economico https://economi.co/mcp
```

Other hosts: swap `claude mcp add` for that host's MCP-install command — the endpoint is the
same. **New emails create a business; known emails log in** — both go through the same
browser page, so there is no separate signup step to run.

### Verify

After the browser flow completes, confirm the connection from inside an MCP session by
listing tools and reading the books:

- Call `tools/list` over `/mcp` — you should see the money-loop tools (`create_invoice`,
  `send_invoice`, `record_payment`, `receive_bill`, `approve_bill`, `summarize_revenue`, …).
- Call `summarize_revenue` for the current month, or `get_invoices` with no filter — a clean
  `200` with an (empty is fine) result confirms you're authenticated against a real business.

If `tools/list` 401s, the OAuth flow didn't finish — re-run the add command and complete the
browser consent.

---

## Path B — CLI

The CLI is a thin shell over the REST API. Every command emits JSON to stdout; errors go to
stderr with a non-zero exit. Requires Node 20+ (Linux, macOS, Windows).

### Install

**Persistent install** (recommended for long-lived agents):

```bash
npm install -g @economico/cli
economico --version
```

**Ephemeral via npx** (no global state — good for sandboxed agents):

```bash
npx -y @economico/cli@latest <command>
```

### Log in (handles signup transparently)

Every business is owned by an email. Run one command:

```bash
economico login
# → Opens the browser, captures the loopback callback (PKCE).
# → Persists access_token + refresh_token to ~/.config/economico/config.json (mode 0600).
# → {"ok":true,"api_url":"https://economi.co","client_id":"cli_pub_…","expires_in":3600}
```

On hosts without a browser (CI, SSH), pass `--no-browser` and the CLI prints the URL for the
user to open manually. The CLI silently rotates the access token via `grant_type=refresh_token`
once on a `401`, so long-running agents don't re-authenticate every hour.

### Per-project credentials (multiple businesses)

When you manage more than one business, sign each into its own project directory instead of
sharing the global config:

```bash
cd ~/companies/acme
economico login --local
# → Writes tokens to ./.economico/config.json (mode 0600) and adds .economico/ to .gitignore.
# → {"ok":true,"api_url":"https://economi.co","client_id":"cli_pub_…","config_path":"…/acme/.economico/config.json",…}
```

Every later command run from `~/companies/acme` (or a subfolder) resolves that local config
automatically; do the same in `~/companies/globex` and the two never collide. Resolution
precedence is: `ECONOMICO_CONFIG_FILE` / `--config <path>` → the discovered local
`.economico/config.json` → the global `~/.config/economico/config.json`. The local file holds
real tokens, so keep it gitignored (the `--local` login does this for you).

### Verify

```bash
economico revenue summary --from 2026-01-01 --to 2026-12-31   # any 2xx → connected
economico accounts list --human                               # see the chart of accounts
```

`economico skill` re-prints the connected deployment's full guide. `economico agent-context`
emits the entire command surface (every command, flag, type, enum) as versioned JSON — read
it before guessing flags. `--help` works at every level (`economico <command> --help`) and is
the canonical source of truth for parameters.

---

## Unattended / headless agents (confidential clients)

For hosted agents that run without a human at a browser (CI jobs, server workers, scheduled
billing), register a confidential OAuth client and authenticate with RFC 7523
`private_key_jwt` instead of the browser flow.

1. Generate an RSA or P-256 EC keypair locally (`openssl genrsa` / `openssl ecparam`).
   Publish only the JWKS; keep the private key secret.
2. From an already-authenticated session (MCP or CLI), call `create_oauth_client` with
   `{ name, jwks }`. The response includes `client_id`, `token_endpoint`, `resource`, `issuer`.
3. The agent signs a short-lived JWT assertion (`alg=RS256` or `ES256`,
   `iss = sub = <client_id>`, `aud = <token_endpoint>`, fresh `jti`, `exp ≤ 60s`) and POSTs to
   `/oauth/token` with `grant_type=client_credentials` to receive a short-lived access token.
4. Use that access token as the `Bearer` on `/mcp` (or REST) requests.

Companion tools: `list_oauth_clients`, `revoke_oauth_client`.

Discovery endpoints:

- `GET https://economi.co/.well-known/oauth-protected-resource`
- `GET https://economi.co/.well-known/oauth-authorization-server`
- `GET https://economi.co/.well-known/jwks.json`

Spec: <https://github.com/modelcontextprotocol/ext-auth/blob/main/specification/draft/oauth-client-credentials.mdx>

---

## You're connected — what next

Once `tools/list` (MCP) or `economico accounts list` (CLI) returns cleanly, the books are
live and authenticated. Hand off to the task-specific cfo-skill:

- **invoicing & billing** → `create_party` → `create_contract` → `create_obligation` →
  `create_invoice` → `send_invoice` → `record_payment`
- **expense tracking** → `receive_bill` → `approve_bill` → `pay_bill`
- **reporting** → `summarize_revenue`, balance sheet / P&L, `get_invoices`, `get_bills`
- **pricing** → `preview_platform_fees`, obligation pricing definitions

For the full money model, chart of accounts, and tool catalogue, run `economico skill` (CLI)
or `tools/list` over `/mcp`, which is always authoritative for the connected deployment.
