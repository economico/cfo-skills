# Economico MCP plugin

Connect a coding agent to Economico's remote MCP server at
[`https://economi.co/mcp`](https://economi.co/mcp). Economico is the finance
function for founders, run by your own agent: contracts, then invoices, then
revenue recognition, plus bills and payments, in exact double-entry books.

## Example prompts

- "Add Juniper Bio as a customer."
- "Juniper Bio signed our order form for a $499 onboarding package. Record it and give me the invoice to send."
- "Record a $120 bill from Hetzner for server hosting."
- "Summarize this month's recorded invoices, payments and open balances."
- "Using our recorded cash and spending history, estimate burn and runway. State the assumptions."

## Connect

On first use the host opens Economico's sign-in. Sign in with email or Google,
create a company or pick an existing one, and approve what the agent may do.
Free for a single founder on Solo; no card required.

## Safety and disclosure

- Every tool call goes to `https://economi.co/mcp` (Economico). The server uses
  OAuth with dynamic client registration; tokens are scoped to the business you
  approve.
- Economico records evidence the agent provides. It does not move money, send
  invoices by email, sign agreements, file taxes, or read your bank or codebase
  on its own. Code, Stripe and email access require separate authorization in
  your agent if you use those tools.
- Burn and runway depend on the records and assumptions supplied.
- Privacy: https://economi.co/privacy/
- Terms: https://economi.co/agreements/
- Support: https://github.com/economico/cfo-skills/issues or support@economi.co

## Hermes

Install from the catalog once listed: `hermes plugins install economico`.
Until then, add the remote server in `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  economico:
    url: "https://economi.co/mcp"
    auth: oauth
```

Setup guide: https://economi.co/agents/hermes/
