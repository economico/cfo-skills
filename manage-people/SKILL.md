---
name: manage-people
description: "Manage People arrangements and payroll in Economico: record direct employees, EOR, PEO and contractors; book provider payroll runs, funding, returns and corrections; verify people_costs and named service seats."
---

# Manage people and payroll

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; confirm the target business before writing). For recruiting pipelines, candidate
stages and closing new hires, use [hiring](../hiring/SKILL.md).

## What this skill owns

1. **Keep one person across arrangements:** Record each worker as one party across direct
   employment, EOR, PEO or contractor arrangements (`people.record_arrangement`,
   `people.set_membership`) without duplicating parties or double-counting costs.
2. **Record payroll and provider runs once:** Model the payroll provider (Gusto, Deel, Rippling,
   EOR/PEO) and book wages, employer taxes, benefits, funding debits, returns and corrections
   cleanly from source run reports. Read [people and payroll](references/people-and-payroll.md).
3. **Connect people to seat usage and verify:** Tie named tool seats back to vendor subscriptions
   via [vendor usage](../manage-vendors/references/vendor-usage.md), and verify `people_costs`,
   payroll payables and bank settlements via [payments](../economico/references/payments.md) and
   [verify](../economico/references/verify.md).

## References

- [People and payroll](references/people-and-payroll.md) — provider discovery, keeping one person
  across arrangements, recording payroll once, funding/returns/corrections, verification
- Vendor contracts & seats: [vendor contracts](../manage-vendors/references/vendor-contracts.md),
  [vendor usage](../manage-vendors/references/vendor-usage.md)
- Shared foundations in `economico`: [payments](../economico/references/payments.md),
  [accounts](../economico/references/accounts.md), [evidence](../economico/references/evidence.md),
  [verify](../economico/references/verify.md)
