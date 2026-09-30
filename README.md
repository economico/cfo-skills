# Economico skills

Agent skills for keeping a founder-run company's books on [Economico](https://economi.co), a
double-entry ledger that your own agent operates. Install them in the agent you already use
(Claude Code, Codex, Hermes, OpenClaw, or anything that reads a skills directory), then ask it to
set up your books.

## Skills

| Skill | What it does |
|---|---|
| `economico` | Connects Economico and models your business in it: reads your product's code, Stripe and email to learn what you sell, to whom, and what it costs to run, records plans, customers, vendors and receipts, and proves the books with reports. Your agent picks it up when you ask about your books. |
| `add-customer` | `/add-customer`: add a customer with a contract on one of your plans. |
| `send-invoice` | `/send-invoice`: bill a customer and get the invoice to send. |
| `record-payment` | `/record-payment`: record a customer's payment against an invoice. |
| `record-vendor-invoice` | `/record-vendor-invoice`: record a bill a vendor sent, with the invoice as evidence. |
| `record-receipts` | `/record-receipts`: find vendor receipts and bills in your email and record them. |
| `investor-update` | `/investor-update`: draft this month's investor update from the books. |

The slash-command skills are shortcuts you type. Each hands your request to the `economico`
skill, so install them together.

## Install

Any agent that reads `~/.agents/skills/` (Claude Code, Codex, Hermes, OpenClaw):

```sh
npx skills add economico/cfo-skills
```

Claude Code, as a plugin:

```sh
/plugin marketplace add economico/cfo-skills
/plugin install economico@cfo-skills
```

Then ask your agent to set up Economico. The `economico` skill connects it (MCP or the
`@economico/cli`) and walks you through the first session. Try it on a sandbox business before
your real one.

## Versions

Releases are tagged `vX.Y.Z` and described in [CHANGELOG.md](./CHANGELOG.md). A major version
removes or renames a skill; a minor one adds a skill, a reference or a recipe; a patch fixes
wording. `v0.1.0` and the `go-era` branch keep the earlier skill set, written for Economico's
previous tools.

This repository is published from Economico's own repository, where the skills are developed and
evaluated. Please open issues here; changes are made there and published.

## License

[MIT](./LICENSE)
