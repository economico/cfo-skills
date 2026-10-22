# Native books and indicative currency totals

Economico's shared reports return native recorded amounts and, where supported, a complete
`indicative` view in the reporting currency. The report service supplies rates from its shared
store. Agents consume these returned values; they do not fetch a provider, convert money or
combine native ratios themselves.

## Read and describe the result

Omit `currency` for the default native-plus-indicative report. An explicit currency requests a
native view and can suppress the additions. Keep these meanings separate:

- Booked functional revenue and expense use their frozen transaction rates. Do not convert them
  again or add the original invoice amount as more revenue.
- Native monetary positions and supported historical native-only flows remain available in
  their own currencies. Show every relevant currency, including a negative cash balance.
- Agreement-derived recurring values are estimates of supported fixed recurring obligations,
  not recognized revenue. Use returned indicative MRR, ARR, waterfalls, cash, burn, runway and
  ratios together; never add runway months, customer counts or percentages across currencies.

Label the returned convenience view **indicative**, state its currency and cutoff, and retain
native detail. Rates identify their exact numerator/denominator, requested day (`requestedDate`
or `requested_date`), observation `date`, `source`, optional `provider`, and `settled` status.
An observation can precede a weekend cutoff. Historical snapshots use their own cutoff rates;
FX movement can change recurring growth and retention without a contract change. Retain the
report's basis and rounding disclosures.

If `missing_rates` is present, say **combined total unavailable**, naming every pair and day.
Show native figures and any independent known facts; do not turn missing into zero, calculate a
partial total, reuse a different month's rate, or infer a missing runway or fee. This also
applies when the source times out. Unsupported inputs and incomplete books stay explicit even
when every rate is available. Keep segmented income statements labelled by period and currency.

`aging` preserves every native group, the undated bucket and a complete indicative total.
`balances.payment_accounts` separates bank/wallet cash, card debt and related-party funding.
Card debt and an owner's advances are not company cash. Scale native amounts using returned
ledger `decimals` when supplied, otherwise the named currency's scale; amounts remain integer
strings until display.

## Optional payment price check

After both executed amounts have been independently evidenced, an optional read is:

```json
{ "action": "run", "kind": "fx_rate", "base": "CHF", "quote": "USD", "date": "2026-09-30" }
```

Call `reports` with that object. Omitting `quote` selects the business's functional currency;
omitting `date` requests the runtime's current day. This read records nothing. Use the returned
exact fraction and provenance only to describe whether the independently evidenced executed
exchange differs from the indicative market observation. Compare exact rational amounts with
the currencies' scales, without floating-point money arithmetic. A difference may reflect
execution timing, spread or fees; do not assign a cause without evidence.

Neither amount may be derived, replaced or "corrected" with this market rate. If one amount is
missing, request its bank/processor evidence. Do not post a guessed FX difference or fee.
The payment command takes the two evidenced amounts and owns any resulting booked gain/loss;
[payments](payments.md) describes transfer and settlement fields. A missing rate only makes the
optional price comparison unavailable; it does not invalidate independently evidenced amounts.
