---
name: setup-company
description: "Set up a company's profile, legal entity and incorporation, bank accounts, company cards and crypto wallets, founder equity, cap table, LLC/partnership capital accounts, owner draws, distributions and year-end allocations in Economico."
---

# Set up company, accounts and ownership

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; fetch recipes via `catalog {action: "recipe", name}`; confirm the target business
before writing). For active fundraising pipelines and closing SAFEs or rounds, see
[fundraise](../fundraise/SKILL.md).

## What this skill owns

1. **Company profile & legal identity:** Update profile description/URL, record legal form and
   jurisdiction changes or incorporation in place (`business.incorporate` /
   `business.change_legal_form`, owner-only with formation documents).
2. **Financial accounts, cards & wallets:** Register checking/savings accounts, company cards,
   payment processors and watch-only crypto wallets with `accounts.register`, plus card statement
   settlement and opening balances.
3. **Ownership & cap table:** Establish actual owners and units from formation documents (never
   guessed from cash received): corporation founders' shares, single-member and multi-member LLC
   units, project owners without issued units, cash contributions, capital distributions, owner
   draws and year-end profit/loss allocations. Read `business {action: "get"}` for the legal form's
   recommended ownership recipes and fetch each via `catalog {action: "recipe", name}`.

Read [company setup](references/company-setup.md) before recording any legal identity, financial
account or ownership fact.

## References and recipes

- [Company setup](references/company-setup.md) — profile, financial accounts, cards/wallets, legal
  identity, founders' shares, LLC/project ownership, draws, distributions and allocations
- Shared foundations in `economico`: [accounts](../economico/references/accounts.md),
  [payments](../economico/references/payments.md),
  [indicative currency](../economico/references/indicative-currency.md),
  [evidence](../economico/references/evidence.md), [verify](../economico/references/verify.md)
- Fetch recipes via `catalog {action: "recipe", name}`: `corporation-founders`,
  `single-member-llc`, `llc-members`, `project-owner`, `owner-cash-contribution`,
  `capital-distribution`, `safe`, `financing-commitment`, `financing-conversion-repayment`
