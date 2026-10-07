# Company setup

The minimum a model needs from the company itself: a profile, the accounts money moves through,
and (when there are owners and investors) the ownership contracts. Legal identity was recorded at
signup; change it only with the owner and the filing in hand.

Use [capital-distribution](recipes/capital-distribution.json) for an evidenced return of
an existing holder's capital. The approved `capital_distribution` exercise creates a
payable; record the bank payment separately against that claim. Select the holder's actual
capital account and retain both authorization and bank evidence. This does not allocate
profit, create an expense, redeem units or permit spending another holder's capital.

## Profile

`business.update` with a one-line `description` (280 characters at most) and the `url`. Write it
straight away from what you learned in discovery; the founder can correct it. `business
{action: "get"}` shows the rest: legal name and form, jurisdiction, functional currency, fiscal
year end, and, under `ownership`, the recipes (ownership setups) that apply to this legal form.

The functional currency was chosen at signup. Check it before the first entry: while the book is
still empty, the owner can correct it with `business.change_functional_currency`
(`fromCurrency`, `toCurrency`, `reason`). Once anything is recorded it is fixed. A company that
moves to another economy is set up as a new business in that currency, not converted. Money in
other currencies is fine either way: each amount lives in its own currency's ledger.

## Financial accounts

Register one financial account per real place money sits, before anything pays or is paid. First
register each bank, card issuer, wallet provider or processor as a party, then link its account
with `providerPartyId`. The account registration classifies that party as a provider. Ask for
the last four digits and network when they identify the account; never record the full number.

```json
{ "name": "parties.create", "input": { "partyId": "mercury", "name": "Mercury" } }
{ "name": "accounts.register", "input": { "financialAccountId": "bank_mercury_checking", "name": "Mercury checking", "currency": "USD", "kind": "checking", "providerPartyId": "mercury", "last4": "1234", "network": "ACH" } }
{ "name": "parties.create", "input": { "partyId": "ramp", "name": "Ramp" } }
{ "name": "accounts.register", "input": { "financialAccountId": "card_ramp", "name": "Ramp card", "currency": "USD", "glAccountCode": 2190, "kind": "card", "providerPartyId": "ramp", "last4": "8570", "network": "Visa" } }
{ "name": "parties.create", "input": { "partyId": "stripe", "name": "Stripe" } }
{ "name": "accounts.register", "input": { "financialAccountId": "stripe_balance", "name": "Stripe balance", "currency": "USD", "kind": "other", "providerPartyId": "stripe" } }
```

- The kind picks the account when `glAccountCode` is omitted: a bank account posts to `1110`,
  `kind: "card"` to `2190` card payable, `kind: "wallet"` to `1310`. A card on `1110` or `1310`
  is refused: a card is a liability, never cash.
- Checking and savings at the same bank are two accounts. One account reachable by several rails
  is one account.
- These are bookkeeping records. Economico does not connect to them, read balances or move money.
- When the provider's fee agreement is known, create its vendor contract as described in
  [provider services](payments.md#provider-services) and link it with
  `providerContractId`. The account and contract remain separate records.
- Register each payment instruction with `payment_endpoints.register`: the account or
  counterparty party, its payto/CAIP-10 URI, and the supported direction, rail and asset.
  Card collection methods attach to the account without a URI. Deactivate a retired
  instruction with `payment_endpoints.update` and register its replacement.
- **Payment details.** When the founder's mail or documents give an account's receiving
  details, register them on that account so the workspace can show and copy them. A US bank
  account is `payto://ach/{routing}/{account}`, an IBAN account `payto://iban/{BIC}/{IBAN}`
  (RFC 8905), a wallet its CAIP-10 id `{namespace}:{chain}:{address}` (`eip155:8453:0x…` for
  Base) with an `onchain` method whose asset is the token's CAIP-19 id
  (`eip155:8453/erc20:0x8335…` for USDC on Base). Copy the numbers exactly as given; never
  guess one, and never record a card number. A card receives nothing, so it has no URI.

```json
{ "name": "payment_endpoints.register", "input": { "paymentEndpointId": "mercury_ach", "financialAccountId": "bank_mercury_checking", "uri": "payto://ach/091311229/202500001234", "label": "USD details", "methods": [{ "direction": "in", "rail": "ach", "asset": "USD" }, { "direction": "in", "rail": "fedwire", "asset": "USD" }] } }
{ "name": "payment_endpoints.register", "input": { "paymentEndpointId": "usdc_base", "financialAccountId": "wallet_usdc_base", "uri": "eip155:8453:0x7a3f9c2e41b8d06a5f1ce8b277d43a90b6e1c91e", "label": "USDC on Base", "methods": [{ "direction": "in", "rail": "onchain", "asset": "eip155:8453/erc20:0x833589fcd6edb6e08f4c7c32d4f71b54bda02913" }] } }
```

- **The account's look.** The workspace draws each account as what it is: a card in its
  issuer's color, a bank account under its bank's band, a wallet in its token's color. Research
  the issuer (the bank, the card issuer — PostFinance, Mercury, Brex — or the token, such as
  Circle for USDC): its brand or press page names the primary color. Set it as `brandColor`
  (`#RRGGBB`) when you register the account, or later with `accounts.update`. When you can
  fetch the issuer's logo as image bytes (a PNG, JPEG or WebP from its press kit, or its
  `apple-touch-icon`; never SVG; at most 256 KiB), receive it with `documents.receive` kind
  `image` and set the returned `documentId` as `logoDocumentId`. Reuse one logo document for
  every account at the same issuer. If you cannot find either, leave them unset rather than
  guess; `null` clears a wrong one.

```json
{ "name": "documents.receive", "input": { "kind": "image", "contentBase64": "iVBORw0KGgo…", "title": "Mercury logo" } }
{ "name": "accounts.update", "input": { "financialAccountId": "bank_mercury_checking", "brandColor": "#1F2033", "logoDocumentId": "doc_…" } }
```
- `payment_routes {action: "quote"}` ranks the active methods that have a matching fee
  term in the account's provider contract. It returns `methodId` and exact minor-unit
  cost; with `party_id`, outbound routes also need that party's matching receiving
  instruction. It is a read, not a payment. Use `payments.process` with the account, rail and explicit gross/net evidence for
  a supported priced provider service; do not pass payment facts into a vendor activity.
  See [payments](payments.md).
- An evidenced owner cash contribution uses the [cash contribution flow](#cash-contributions-without-new-units),
  including when it establishes the bank's initial balance. Record the contribution and its
  independent payment with their source identities; a journal would lose the payment/claim link.
  An opening `journal.post` is only for an agreed carry-forward of pre-existing books, not a
  substitute for supplied transaction facts. Do not replay those carried-forward transactions.

## Legal identity (owner only, with evidence)

Changing the legal name, legal form, jurisdiction or register number, the fiscal year, the
accounting framework, or registering for sales tax are separate commands
(`business.change_legal_name`, `business.change_legal_form`, `business.set_registration`,
`business.change_fiscal_year`, `business.elect_accounting_framework`,
`business.register_sales_tax`). Each needs the owner's interactive session and a
`source_document_id` naming the filing or certificate. Never guess a registration number: ask for
it exactly as printed. A headless agent cannot run these.

These commands appear in `catalog list` only when the business's owner approved this
connection: every connection gets full access, but only up to the approving person's own
permissions. If they are missing, ask the owner to connect the agent themselves; never try to
work around it.

## Ownership

The corporation, LLC/member and project-owner recipes below use priced rights. Discover the
supported ownership right and evidenced authorization, subscription, vesting or claim-conversion
consequence in the live catalog. Record capital rights separately from independent funding;
reimbursement needs only its identified owner claim. If the required ownership family cannot
be represented, preserve the documents and report the gap rather than inventing an invoice or
silently changing the agreement. The profile and financial-account procedures above remain
current.

### Cash contributions without new units

Follow [owner cash contribution](recipes/owner-cash-contribution.json), replacing its synthetic
evidence with the actual authorization and deposit. When the source explicitly authorizes an owner capital contribution but specifies no new share
issuance, use an ownership right with `direction: "provide"`, a priced money amount and
explicit `classification.capitalAccount` (other contributions use 3120). Declare an exercise
consequence `capital_contribution` with matching money consideration; no units ledger or
quantity is required. Accept the agreement with its evidence, then `contracts.record` its
`exercise` with economic evidence and a distinct source fact. This establishes an investment
receivable and capital. Record the observed bank deposit with `payments.record`, direction
`in`, allocating to the returned claim/component. Preserve the deposit's source identity.
Do not treat an unexplained deposit or a loan as a contribution, and do not invent shares.

### Evidenced convertible financing commitments

Use [financing commitment](recipes/financing-commitment.json) when the signed instrument
establishes an unconditional funding obligation, or economic evidence proves its funding
condition was satisfied. Declare a financing right and a `financing_commitment` exercise with
matching monetary consideration. Its monetary classification has no capital, revenue, expense
or tax account. Acceptance alone records no obligation. The evidenced exercise establishes
receivable 1120 and convertible financing liability 2240; a separate incoming payment settles
the receivable. Preserve the instrument's actual classification and source IDs. Do not use this
for an equity subscription, an unevidenced promise, or an ordinary trade invoice.

The claim report identifies funding receivable 1120 and financing debt 2240 separately.
They share a claim ID, so preserve the full component and choose by direction; never use
claim ID alone. Follow [conversion and repayment](recipes/financing-conversion-repayment.json)
for an evidenced debt-to-equity exercise and a separate bank repayment of the remainder.
Use the actual agreed equity account and unit quantity if stated; do not invent shares.
`cap_table` includes recognized outstanding financing, even before funding, under its original
agreement. Funding cash and unconverted debt are different balances. Frozen
commitments recorded before liability claims were identified remain readable but do not gain an invented debt claim.

### Close existing owner draws

For accumulated draws already recorded on 3300, bind a `close_draws` consequence to the
member's ownership right and declared period schedule. Select that holder's actual capital
account in the right's monetary classification. Record `period_close` only after the period
ends, with approval, economic evidence and a typed instant `completeThrough` fact covering
the exclusive end. This reclassifies the draws into capital; it does not pay the owner again
or allocate profit. Read `capital_accounts` and confirm total capital and cash are unchanged.
Prior closing credits consume the oldest draws first, even when recorded at or after the
period boundary; new draws cannot replenish a closed period. When multiple period rules match the same window, select the declared `consequenceKey`. Do not create a separate
bill, payment or "closing activity" to make the totals work.

### Allocate retained profit or loss

Bind `profit_allocation` to the member's ownership right and completed period, with the
actual monetary and membership ledgers. The holder must already own units; allocation
never admits a member or issues units. Supply an approved nonnegative amount in the
monetary ledger's unit. The rule defaults to profit; declare `result: loss` for losses.
Use `period_close`, the exact schedule window, and its final instant as `effectiveAt`
(one millisecond before the exclusive end). Record only after completion and the due date,
with an instant `completeThrough` fact and acceptance/economic evidence. Select
`consequenceKey` if multiple period consequences match. A period cannot cross a fiscal year.

The kernel bounds the allocation against established retained result in that fiscal year,
including later allocations. It does not determine allocation ratios or tax treatment from
membership units. Read `capital_accounts` to verify capital and undistributed earnings,
and the income statement to confirm no second revenue or expense. Distributing cash is a
separate approved distribution right and independent payment. Schema 12 is required.

The [SAFE example](recipes/safe.json) now uses a priced financing right, evidenced purchase
conditions, and independent bank funding. It preserves the financing liability without
inventing shares, equity, revenue or a commercial invoice. Its evidence is explicitly synthetic;
replace it with the signed instrument and actual bank record.

### Member draws and year-end allocation

Follow [LLC members](recipes/llc-members.json) for two subscriptions to one operating agreement,
independent funding, a separate customer agreement and approved owner draws. The `owner_draw`
exercise creates a distribution payable and drawing balance; pay the identified claim separately.
At year end record the evidenced allocation amount and `completeThrough`, then close accumulated
draws into member capital. The example's $40,000 result at an agreed 50/50 split yields $20,000
per member; do not infer allocation percentages from who paid cash. Retain the operating agreement
and allocation evidence. Draws do not recognize expenses or redeem membership units.

### A project owner without issued units

Follow [project owner](recipes/project-owner.json) for capital contributions, repayable advances,
owner-paid supplier costs contributed to capital, draws and loss allocation. Its vendor receipt
is explicitly synthetic; replace the supplier and evidence with the real source. The vendor
purchase remains a separate agreement. Convert the identified owner funding claim to capital
with `convert_claim`; do not invent an invoice or move cash again.

Declare `liability: "owner_advance"` on repayable owner financing, then fund its receivable and
repay its owner-liability component independently. For profit/loss allocation when the agreement
has no issued units, declare `ownershipBasis: "agreement"`; retain ownership and allocation
evidence, and supply the agreed allocation amount. Do not manufacture shares to satisfy a report.

### Historical ownership reference

Founders' shares, SAFEs, LLC members and project owners are contracts too, with the same
activity machinery and dedicated reports (`cap_table`, `capital_accounts`). Model them in every
first session, for every company that has owners, including a sole owner and owners who have put
in no cash yet: an empty cap table says nobody owns the business. `business get` names the
recipes for the legal form under `ownership`. The rules that matter:

- Founders are individual parties, never one "Founders" party.
- Give every holder its role when you create it, because no contract sets one: a founder, an
  LLC member or partner who runs the business, and a project's owner are
  `categories: ["founder"]`; a SAFE or priced-round investor, or a parent company that funds a
  project, is `["investor"]`; an employee granted restricted stock is `["employee"]`. A party
  without one is listed under "Other".
- A SAFE is financing (`fund_liability`, 2240) until it converts; it is not equity and not
  revenue.
- Cash received never implies a share count; the share purchase agreement does.
- The quantities come from the formation documents (the certificate, the stock purchase
  agreements, the operating agreement) or from the founder. When none of them says, ask; do not
  leave the cap table empty in silence.
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
4. **The subscription** exercises the ownership right with `issue_units`. Its money
   consideration creates one investment receivable; `capitalAllocation` splits the evidenced
   par amount into share capital (3100) and the premium into paid-in capital (3110). Set
   `restricted: true` for the grant. Record the actual bank funding independently with
   `payments.record`, allocated to that claim. Issuance does not prove payment.
5. **Vesting** uses `vest_units` period consequences of that same right: a one-year cliff
   followed by a monthly schedule. `grantConsequence` names the original issuance; it must
   identify exactly one grant. Give both schedules `serviceEvidence: "elapsed_time"` only
   when the agreement authorizes vesting through elapsed service. Put the evidenced tranche
   quantities in terms. When the grant is not a multiple of the monthly tranche, the period
   that leaves less than a full tranche vests exactly what remains (1000 at 334 a month vests
   334, 334, 332), so round the tranche up. Restricted units are outstanding from
   issuance. The timer records elapsed tranches through the ordinary economic command;
   `activity_plan` previews them as `fixed`, keeps missing evidence contingent and refuses
   projected over-vesting. A manual recording uses the exact period and its final instant,
   with economic and service evidence, after the due date.
6. Check with `reports {action: "run", kind: "cap_table"}`: authorized, outstanding, and each
   founder's quantity, vested and unvested units. Show it to the founder as a table.

Shares without vesting use `issue_units` with `restricted: false` and no vesting consequences. Units
issued for no consideration use an explicit zero price and matching zero consideration, as in
the single-member LLC below.

### A single-member LLC

The common case for a solo founder (a Delaware LLC from Stripe Atlas, for example): one member
holds the whole company, often issued for no capital at all. Follow
[single-member LLC](recipes/single-member-llc.json):

1. Store the formation evidence (the certificate of formation, the operating agreement, or the
   formation service's confirmation) and set the legal form with `business.change_legal_form`
   (`llc` and its jurisdiction) if `business get` does not have it yet.
2. One operating agreement identifies the member's ownership right and the membership units
   explicitly stated by the agreement. Accept it with evidence, then record its declared
   authorization and subscription consequences. Do not invent units from a percentage or
   infer a cash contribution from admission to membership.
3. **Units for $0.00.** When the agreement issues the units for no capital contribution (the
   founder's IP assigned under a separate CIIAA is not one), price the right explicitly at zero
   (`price: {treatment: "priced", calculation: {type: "fixed", amount: "term:contribution"}}`
   with the term `contribution` = `"0"` in USD) and give `issue_units` the matching money
   consideration (`amount: "term:contribution"`). Record the `exercise` with the operating
   agreement as `economic` (and any other declared) evidence. The units are issued with no
   investment claim, receivable or capital posting: `claimIds` is empty and the member's
   capital is $0. An unknown or implicit price is refused; never stand in a nominal amount.
4. Record cash contributions, financing advances, conversions of owner claims and approved
   drawings through their own evidenced consequences; record each bank movement separately.
   An approved `owner_draw` creates a payable. `close_draws` later closes the drawing balance.
5. Check `cap_table` and `capital_accounts` against the actual agreement and recorded funding.

### Other holders

The same shape, one party and one contract per holder, with the recipe `business get` names:

| Holder | Recipe |
|---|---|
| A SAFE, held as financing until it converts | [safe](recipes/safe.json) |
| An LLC's or partnership's members: capital, draws, year-end allocation | [llc-members](recipes/llc-members.json) |
| A project's owner: investment, repayable advances, draws, year-end allocation of a loss | [project-owner](recipes/project-owner.json) |

At year end, an LLC's, partnership's or project's result goes to the owners' capital: a profit
through `profit_allocation` with explicit profit/loss treatment and each owner's evidenced
allocation amount, so `capital_accounts` reconciles the remaining undistributed result. The ledger refuses an allocation larger
than the profit or loss not yet allocated.

Discover each priced-right consequence in `catalog` before adapting the validated recipes.

For a priced share subscription with evidenced par and premium amounts, the `issue_units` consequence accepts `capitalAllocation: {shareCapital: "term:parAmount", sharePremium: "term:premiumAmount"}`. Bind both amounts from the agreement in the same currency as the subscription price; they must sum exactly to the consideration. Use share-capital classification. Do not infer par from an ownership percentage or record another subscription to represent premium. Funding remains an independent payment against the single investment claim.
