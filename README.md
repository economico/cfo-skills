# cfo-skills

Agent skills that turn a coding agent into a working CFO on
[Economico](https://economi.co) — an agent-native double-entry general ledger.
Install, run `setup-economico`, then price, contract, invoice, pay bills, analyze,
forecast, and report. Economico is the system of record (MCP/CLI); these skills
are the playbooks (double-entry reasoning, period close, unit economics).

**Fit:** new SaaS/consulting company, no books to migrate, founder-run agent.
**Constraint:** write access is YC-founder gated until `verify_yc` — see
https://economi.co/skill.md.

## Skills

| Skill | What it does |
|-------|--------------|
| [`setup-economico`](./setup-economico) | Connect & authenticate Economico (MCP or `@economico/cli`); verify the connection before doing CFO work |
| [`company-setup`](./company-setup) | Guided first-run setup once connected — legal-entity profile, named bank/wallet accounts, and the cap table (share classes, founder issuance, early SAFEs) — then hand off to pricing, contracts, and expense-tracking |
| [`payment-rails`](./payment-rails) | Model how money moves in and out — register each account's identifiers (payto / CAIP-10) and priced in/out methods, account-level card/Flow collection, and counterparty payment instructions; rank routes by cost or speed; post route fees to the ledger |
| [`financial-analyst`](./financial-analyst) | Read-only analysis of the books — reports plus ratios, trends, aging, concentration, burn, and runway, drilling into the journal when needed |
| [`expense-tracking`](./expense-tracking) | Ramp-style AP: find receipts/invoices in email and record them as bills, building each vendor's party → contract → obligation spine so every line maps to an account and feeds unit economics |
| [`pricing`](./pricing) | Create a customer-facing `pricing.md` and map its charges into Economico contracts, obligations, account codes, and platform-fee previews |
| [`creating-contracts`](./creating-contracts) | Create order forms, choose standard terms, and set up customer contracts plus billable obligations before invoicing |
| [`invoicing`](./invoicing) | Create, send, void, and reconcile customer invoices, preferring active contracts and obligation-linked lines |
| [`investor-reporting`](./investor-reporting) | Read-only investor updates and board metrics: MRR, ARR, ACV, NDR, burn, runway, default-alive, margin, concentration, and model-specific KPIs |
| [`cpa`](./cpa) | Read-only CPA books audit: tie out the statements and surface misstatements, misclassifications, missing accruals, and GAAP departures as ranked findings, each with evidence and the correcting entry to recommend |
| [`forecasting`](./forecasting) | Project future cash, runway, MRR/ARR, and committed inflows/outflows — running what-ifs (price change, new deal, churn) and forward-dated draft entries inside disposable planning scenarios, so the real ledger stays untouched |

## Planned skills

| Skill | What it does |
|-------|--------------|
| `reporting` | Produce P&L, balance-sheet, AR/AP, and revenue summaries for a period |

This list is a starting point, not a contract — see
[`CLAUDE.md`](./CLAUDE.md) for how to add or change skills.

## Using with Economico

Economico is the ledger these skills operate on. Connect it as an MCP server so
the agent can read and post to the books:

```sh
claude mcp add economico https://economi.co/mcp
```

Claude Code opens the OAuth browser flow on first use (email + 6-digit code).
See the [Economico README](https://economi.co) for the MCP tool surface
(`create_invoice`, `record_payment`, `receive_bill`, `summarize_revenue`, …)
that these skills build on.

## Installation

These are plain `SKILL.md` folders, so they work in any agent that reads a skills
directory.

### `npx skills add` (recommended — works across agents)

```sh
npx skills add economico/cfo-skills
```

Installs into the cross-agent `~/.agents/skills/` directory (plus per-host paths),
so the one command is picked up by Claude Code, Codex, Hermes, and OpenClaw.

### Host-native paths

- **Claude Code** — add the plugin marketplace, then install skills with `/plugin`:

  ```sh
  /plugin marketplace add economico/cfo-skills
  ```

- **Hermes** — add the repo as a tap (its Skills Hub also federates the skills.sh
  listing):

  ```sh
  hermes skills tap add economico/cfo-skills
  ```

- **OpenClaw** — reads `~/.agents/skills/`, so the `npx skills` install is
  discovered automatically.

## Repository structure

```
.claude-plugin/
  marketplace.json     # Marketplace manifest (skills.sh + Claude Code plugins)
<skill-name>/          # Source directory for each skill
  SKILL.md             # Frontmatter (name, description) + skill prompt
  references/          # Supporting playbooks, formulas, account mappings
<skill-name>.skill     # Packaged zip of the source directory (built on release)
README.md
CLAUDE.md              # Authoring + build rules for this repo
LICENSE
```

## License

[MIT](./LICENSE).

## Resources

- [Economico](https://economi.co) — the agent-native ledger these skills drive
- [Anthropic Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
- [skills.sh](https://skills.sh)
