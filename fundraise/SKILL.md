---
name: fundraise
description: "Run fundraising in Economico: design and run investor pipelines for your round, capture investor conversations, close SAFEs, convertible notes and priced rounds in one motion, record funding commitments, conversions, repayments and wires."
---

# Fundraise

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; fetch recipes via `catalog {action: "recipe", name}`; confirm the target business
before writing). For entity formation, founder shares, LLC/project capital accounts and company
bank accounts, use [setup-company](../setup-company/SKILL.md); for monthly reporting to existing
investors, use [investor-update](../investor-update/SKILL.md).

## 1. Research the round and recommend the investor pipeline

Before opening pursuits or defaulting to the generic built-in 3-stage `financing` pipeline
(`open`/`qualified`/`proposed`), check existing pipelines
(`subjects {action: "list", type: "pipeline"}`) and research the business's fundraising context.
Read the current cap table (`reports run cap_table`), company profile (`businesses {action: "get"}`),
`business-model.md`, active targets (`targets {action: "list"}`), and any mailbox threads, decks or
notes about the raise to identify the instrument, stage and investor process:

| Raise type | Pipeline recommendation | Suggested stages (default `weightBps`) |
|---|---|---|
| **Rolling SAFE / convertible note / angels & pre-seed** (default investor pipeline) | Customize `financing` via `pipelines.define` (or define a round-specific key like `pre-seed` when running separate tracks). | `target` (500), `intro` (1000), `first_meeting` (2000), `diligence` (4000), `verbal_commit` (7500), `docs` (9000) |
| **Priced equity round / institutional lead** (Seed, Series A+ with partner vote & term sheet) | Customize `financing` (or define `seed` / `series-a`); tag pursuits with `labels` like `role` (`lead`/`follow`), `type` (`vc`/`angel`/`strategic`), `round`, and `intro_by`. | `target` (500), `intro` (1000), `first_meeting` (1500), `partner_meeting` (3500), `diligence` (5500), `term_sheet` (7500), `docs` (9000) |
| **Grants / non-dilutive / venture debt** | Define a dedicated pipeline (e.g. `grants` or `debt`) so non-dilutive pursuits stay separate from equity/SAFE investors. | `identified` (1000), `application` (2500), `review` (5000), `approved` (8500) |

- If a custom `financing` pipeline is already defined in Economico, use its existing stages unless
  the founder asks to revise them.
- If no custom fundraising pipeline exists yet, recommend the pipeline structure that matches the
  round (stages, weights, useful labels) alongside a live **fundraise round target** (`at_least`
  raised goal on the `financing` pipeline with weighted open-pipeline coverage, recorded
  proposed-first via [plan-and-track](../plan-and-track/SKILL.md) and `targets-fundraise`), explain
  why, and confirm with the founder via
  [asking the founder](../economico/references/asking-the-founder.md) before calling
  `pipelines.define` or activating targets. When the founder gave explicit instructions or is away,
  apply the best-fit recommendation and state the chosen pipeline, stages and targets in the final
  summary.

## 2. The fundraising loop

1. **Track investors in the pipeline:** Create each prospective investor as an uncategorised party
   (`parties.create` with no categories yet), open pursuits with `opportunities.open`
   (`pipelineKey: "financing"` or the agreed round pipeline), move stages with `opportunities.stage`,
   and log passes with `opportunities.lose`. Capture investor meetings and term sheets via
   [conversation intake](../economico/references/conversation-intake.md) and
   [record evidence](../economico/references/record-evidence.md). Read
   [pipelines](../economico/references/pipelines.md).
2. **Close a SAFE, note or round in one motion:**
   - Align `expectedValue` via `opportunities.stage` if the signed check size differs from the
     initial estimate so won pipeline value matches what was signed.
   - Store the signed instrument via `documents.receive`, create the financing contract with
     `partyCategories: {"investor": ["investor"]}`, link it with `opportunities.link`, and accept
     with `contracts.accept` (which wins the pursuit and adds the `investor` role).
   - Immediately record the `exercise` phase (`contracts.record`) for the evidenced commitment
     (`financing_commitment` / SAFE purchase condition) — acceptance alone puts nothing on the cap
     table or balance sheet.
   - Record the bank wire separately via `payments.record` (`direction: "in"`) against the
     identified receivable claim (`1120`) and component (preserving `2240` convertible financing
     liability separately; never use claim ID alone).
3. **Conversions, repayments and cap-table verification:** Follow
   [company setup: financing commitments & SAFEs](../setup-company/references/company-setup.md#evidenced-convertible-financing-commitments)
   for debt-to-equity conversions, repayments and `cap_table` verification, and track round targets
   in [plan-and-track](../plan-and-track/SKILL.md).

## References and recipes

- Shared foundations in `economico`: [pipelines](../economico/references/pipelines.md),
  [conversation intake](../economico/references/conversation-intake.md),
  [payments](../economico/references/payments.md), [evidence](../economico/references/evidence.md),
  [verify](../economico/references/verify.md)
- Cap table, financing commitments & SAFEs:
  [company setup](../setup-company/references/company-setup.md#evidenced-convertible-financing-commitments)
- Fetch recipes via `catalog {action: "recipe", name}`: `targets-fundraise`, `safe`,
  `financing-commitment`, `financing-conversion-repayment`
