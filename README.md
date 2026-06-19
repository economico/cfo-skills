# cfo-skills

Agent skills that turn a coding agent into a working CFO — pricing, expense
tracking, invoicing & billing, forecasting, and reporting.

The skills are designed to run against [Economico](https://economi.co), an
agent-native general ledger. Economico exposes its books primarily over MCP
(Model Context Protocol), so these skills teach an agent *how* to do CFO work —
the playbooks, the double-entry reasoning, the period-close discipline — while
Economico's MCP tools provide the *system of record* underneath.

## Skills

| Skill | What it does |
|-------|--------------|
| [`setup-economico`](./setup-economico) | Connect & authenticate Economico (MCP or `@economico/cli`); verify the connection before doing CFO work |

## Planned skills

| Skill | What it does |
|-------|--------------|
| `pricing` | Model and set usage-based / subscription pricing; preview platform fees |
| `expense-tracking` | Record vendor bills, categorize spend, run accounts payable |
| `invoicing-billing` | Draft, send, and reconcile customer invoices; run recurring billing |
| `forecasting` | Build revenue, cash, and runway forecasts off ledger history |
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

### Option 1: `npx skills add` (recommended)

```sh
npx skills add economico/cfo-skills
```

### Option 2: Claude Code marketplace

```sh
/plugin marketplace add economico/cfo-skills
```

Then install individual skills from the marketplace with `/plugin`.

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
