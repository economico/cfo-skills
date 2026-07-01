# First-run checklist

The ordered setup this skill drives, and which skill owns each later step. Skip
anything already done; confirm before anything posts to the books.

## This skill (`company-setup`)

- [ ] **Connected & un-gated** — `get_business` returns; if not YC-verified,
      `verify_yc(link)` first (every write below is gated until then).
- [ ] **Profile** — `update_business(name, legal_entity_type, jurisdiction,
      description, url)`.
- [ ] **Bank / wallet accounts** — for each: resolve-or-create the provider party,
      then `create_financial_account(...)`. Register checking **and** savings
      separately even in the same currency.
- [ ] **Cap table** — `create_share_class` → `issue_shares` per founder →
      `record_safe` for any early SAFEs → `get_cap_table` to confirm.

## Hand off from here

| Next job | Skill | Starting tools |
|---|---|---|
| Design pricing / write `pricing.md` | **`pricing`** | `preview_platform_fees`, obligation pricing |
| Paper a customer deal / order form | **`creating-contracts`** | `create_contract`, `create_obligation` |
| Bill a customer | **`invoicing`** | `create_invoice`, `send_invoice`, `record_payment` |
| Vendors, receipts, "scan my email for bills" | **`expense-tracking`** | `create_party` → `receive_bill` → `approve_bill` |
| Read the books / runway / investor metrics | **`financial-analyst`**, **`investor-reporting`** | reports, `get_saas_metrics` |

## Not this skill

- **Connecting / authenticating the ledger** → `setup-economico` (the prerequisite).
- Reading or analyzing the numbers → `financial-analyst`.

Rehearse anything uncertain in the seeded **`test`** scenario first (pass
`scenario: "test"`); the real ledger is the default when you omit it.
