# Company setup

The minimum a model needs from the company itself: a profile, the accounts money moves through,
and (when there are owners and investors) the ownership contracts. Legal identity was recorded at
signup; change it only with the owner and the filing in hand.

## Profile

`business.update` with a one-line `description` (280 characters at most) and the `url`. Write it
straight away from what you learned in discovery; the founder can correct it. `business
{action: "get"}` shows the rest: legal name and form, jurisdiction, functional currency, fiscal
year end, and, under `ownership`, the recipes (ownership setups) that apply to this legal form.

## Financial accounts

Register one financial account per real place money sits, before anything pays or is paid.

```json
{ "name": "accounts.register", "input": { "financialAccountId": "bank_mercury_checking", "name": "Mercury checking", "currency": "USD", "kind": "checking" } }
{ "name": "accounts.register", "input": { "financialAccountId": "card_ramp", "name": "Ramp card", "currency": "USD", "glAccountCode": 2190, "kind": "card" } }
{ "name": "accounts.register", "input": { "financialAccountId": "stripe_balance", "name": "Stripe balance", "currency": "USD", "kind": "other" } }
```

- The kind picks the account when `glAccountCode` is omitted: a bank account posts to `1110`,
  `kind: "card"` to `2190` card payable, `kind: "wallet"` to `1310`. A card on `1110` or `1310`
  is refused: a card is a liability, never cash.
- Checking and savings at the same bank are two accounts. One account reachable by several rails
  is one account.
- These are bookkeeping records. Economico does not connect to them, read balances or move money.
- The opening balance is not typed in. It comes from the recorded history (founder investment,
  SAFE funding) or, when the founder is starting mid-stream, from an opening `journal.post`
  with the bank statement as its source document, agreed with the founder.

## Legal identity (owner only, with evidence)

Changing the legal name, legal form, jurisdiction or register number, the fiscal year, the
accounting framework, or registering for sales tax are separate commands
(`business.change_legal_name`, `business.change_legal_form`, `business.set_registration`,
`business.change_fiscal_year`, `business.elect_accounting_framework`,
`business.register_sales_tax`). Each needs the owner's interactive session and a
`source_document_id` naming the filing or certificate. Never guess a registration number: ask for
it exactly as printed. A headless agent cannot run these.

## Ownership

Founders' shares, SAFEs, LLC members and project owners are contracts too, with the same
activity machinery and dedicated reports (`cap_table`, `capital_accounts`). Model them when the
founder asks, or when the brief needs cash that came from them (a SAFE or the founders' share
purchases are where the opening cash came from). The rules that matter:

- Founders are individual parties, never one "Founders" party.
- A SAFE is financing (`fund_liability`, 2240) until it converts; it is not equity and not
  revenue.
- Cash received never implies a share count; the share purchase agreement does.
- Stock options, option pools and tax elections (83(b)) are not modeled; say so.

`business {action: "get"}` returns, under `ownership`, the equity heading and the ownership
reports that apply to the business's legal form.

### Founders' shares in a corporation

Follow [corporation-founders](recipes/corporation-founders.json), with the founders' own
documents in place of its synthetic ones:

1. Store the certificate of incorporation and each signed stock purchase agreement
   (`documents.receive`), and create one party per founder (`categories: ["founder"]`).
2. **Authorize the class once**, on a charter contract evidenced by the certificate: the
   authorized share count from the certificate, on an `issued_instrument` ledger.
3. **One contract per founder** from one purchase template, the founder's shares, par and price
   as term overrides. Bind every founder's contract to the charter's share ledger (its id is in
   the charter contract's `state.ledgers`), so all purchases draw on one authorization; an
   unbound contract gets its own empty ledger and the grant is refused.
4. **The purchase** is one activity: `fund_capital` for par × shares (to 3100), `fund_premium`
   for the rest of the price (3110, often zero when founders buy at par) and `grant_restricted`
   for the shares, recorded when the money arrived, into the registered bank account.
5. **Vesting** is scheduled, never recorded early: a one-year cliff (one period of twelve
   months from the vesting commencement date) and a monthly schedule after it, each `vest` with
   the purchase as its `accountingKey`. Give both schedules `"serviceEvidence": "elapsed_time"`:
   without it every tranche plans as contingent and never vests. Put the per-founder tranche
   sizes in the terms; they must add up to the grant. Restricted shares count as outstanding and
   unvested from the purchase; the ledger's timer records each tranche when its date passes, and
   `reports {action: "run", kind: "activity_plan", contract_id, from, through}` shows them as
   `fixed` before it does.
6. Check with `reports {action: "run", kind: "cap_table"}`: authorized, outstanding, and each
   founder's quantity, vested and unvested units. Show it to the founder as a table.

Shares bought without vesting use `allocate` instead of `grant_restricted` and no vesting
activities.

### Other holders

The same shape, one party and one contract per holder, with the recipe `business get` names:

| Holder | Recipe |
|---|---|
| A SAFE, held as financing until it converts | [safe](recipes/safe.json) |
| An LLC's or partnership's members: capital, draws, year-end allocation | [llc-members](recipes/llc-members.json) |
| A project's owner: investment, repayable advances, draws | [project-owner](recipes/project-owner.json) |

`catalog {action: "describe"}` the effect patterns a recipe uses before adapting it.
