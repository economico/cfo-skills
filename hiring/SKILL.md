---
name: hiring
description: "Run the hiring pipeline in Economico: define hiring pipelines and stages, track candidates and interview context, close accepted offers in one motion with People arrangements and headcount membership, and track hiring targets."
---

# Hiring

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; confirm the target business before writing). For ongoing payroll runs, EOR/PEO
bills, contractor invoices and named seats after someone joins, use
[manage-people](../manage-people/SKILL.md).

## The hiring loop

1. **Define or read the hiring pipeline:** Use `pipelines.define` with `pipelineKey: "hiring"` (or
   the company's chosen key), ordered stages with `weightBps`, and `settlementKinds:
   ["people_arrangement"]` (or inspect existing pipelines with
   `subjects {action: "list", type: "pipeline"}`). Read
   [pipelines](../economico/references/pipelines.md).
2. **Track candidates and interviews:** Create each candidate as an uncategorised party
   (`parties.create` with no categories yet), open a pursuit with `opportunities.open` (labelled by
   `role`, `owner` or `source`, with estimated annual cost or `null` if unknown), and move stages
   with `opportunities.stage`. Attach interview notes or call context with
   [conversation intake](../economico/references/conversation-intake.md) and
   [record evidence](../economico/references/record-evidence.md).
3. **Close an accepted hire in one motion:** When a candidate accepts, record both
   `people.record_arrangement` (direct employee, EOR, PEO or contractor — automatically wins a
   single open unlinked pursuit for that person) and `people.set_membership` (so the person counts
   in headcount / "people in place") together in one sitting.
   - If the person has several open pursuits, link the specific arrangement explicitly with
     `opportunities.link` (`arrangementId`).
   - Employment history that ended before the pursuit opened never wins it; link it explicitly if
     that is the arrangement.
   - Retracting an arrangement (`through <= from`) reopens the linked pursuit so recording the
     right arrangement wins it.
   - Record declined or withdrawn candidates with `opportunities.lose` and the founder's `reason`.
4. **Connect to hiring targets and ongoing payroll:** Track `to_date` hires and `each_period`
   people-in-place goals in [plan-and-track](../plan-and-track/SKILL.md), and hand off payroll,
   EOR/PEO contracts and tool seats to [manage-people](../manage-people/SKILL.md).

## References and recipes

- Shared foundations in `economico`: [pipelines](../economico/references/pipelines.md),
  [conversation intake](../economico/references/conversation-intake.md),
  [record evidence](../economico/references/record-evidence.md),
  [verify](../economico/references/verify.md)
- Ongoing payroll and People arrangements: [manage-people](../manage-people/SKILL.md)
- Fetch the end-to-end hiring recipe via `catalog {action: "recipe", name: "targets-hiring-plan"}`.
