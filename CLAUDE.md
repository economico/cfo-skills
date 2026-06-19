# cfo-skills — Claude Skills Marketplace

This repository contains Claude Code skills for CFO tasks (pricing, expense
tracking, invoicing & billing, forecasting, reporting), distributed both as a
Claude Code plugin marketplace and via `npx skills add`.

The skills operate against [Economico](https://economi.co), an agent-native
general ledger whose primary API is MCP. Skills here encode CFO *workflow and
judgment*; Economico's MCP tools are the *system of record*. When authoring a
skill, assume the agent has the `economico` MCP server connected and prefer its
tools (`create_invoice`, `send_invoice`, `record_payment`, `receive_bill`,
`approve_bill`, `summarize_revenue`, `create_contract`, `create_obligation`,
`preview_platform_fees`, …) over re-implementing ledger logic.

## Repository structure

Each skill has a source directory and, on release, a packaged `.skill` file:

```
<skill-name>/          # Source directory
  SKILL.md             # Frontmatter (name, description) + skill prompt
  references/          # Supporting playbooks, formulas, account mappings
<skill-name>.skill     # Packaged zip of the source directory
```

The marketplace manifest lives at `.claude-plugin/marketplace.json`.

## Adding a skill

1. Create `<skill-name>/SKILL.md` with valid frontmatter (`name`, `description`)
   between `---` delimiters, followed by the skill prompt.
2. Add supporting material under `<skill-name>/references/` as needed.
3. Register the skill in `.claude-plugin/marketplace.json` by appending an entry
   to the `plugins` array:

   ```json
   {
     "name": "<skill-name>",
     "description": "<one-line description>",
     "source": "./",
     "strict": false,
     "skills": ["./<skill-name>"]
   }
   ```

4. Build the `.skill` zip (see below) and update the table in `README.md`.

## Rules

### Keep .skill files in sync

Whenever you modify any file inside a skill's source directory, rebuild its
`.skill` file before finishing:

```sh
zip -r <skill-name>.skill <skill-name>/
```

Never leave a `.skill` file out of sync with its source directory.

### Description length limit

The `description` field in each `SKILL.md` frontmatter **must be under 1024
characters**. After editing a description, verify:

```sh
awk '/^description: >/{found=1; next} found && /^---$/{exit} found{gsub(/^  /,""); printf "%s ", $0}' <skill-name>/SKILL.md | wc -c
```

### Skill correctness checks

When reviewing or editing a skill, verify:

1. **Frontmatter is valid** — `name` and `description` exist between `---` delimiters.
2. **Description < 1024 chars** — see above.
3. **MCP tool references are accurate** — tool names, arguments, and the
   double-entry effects match Economico's current MCP surface. When in doubt,
   check the Economico repo / docs rather than guessing.
4. **No stale information** — if Economico's tools or accounting conventions
   change, update the skill to match.
5. **Manifest is current** — every skill directory has a matching entry in
   `.claude-plugin/marketplace.json`, and the `plugins` array has no dangling
   references.
6. **Zip is current** — the `.skill` file contains exactly the source directory.

### Accounting discipline

These skills move real money and post to real books. Author them to:

- Respect double-entry — every posting balances; never invent one-sided entries.
- Be idempotent — rely on Economico's `external_ref` / composite-key idempotency
  rather than re-posting.
- Prefer reading the ledger (`summarize_revenue`, `get_invoices`, `get_bills`)
  before writing, and confirm before any irreversible or outward-facing action
  (sending an invoice, recording a payment).

### Commit hygiene

- Commit a skill's source directory changes and its rebuilt `.skill` file together.
- Keep commits focused: one skill per commit when possible.
