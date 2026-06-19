# Avoiding double-records and handling repeat charges

Email gets re-scanned and vendors re-send invoices, so the main risk is recording
the same expense twice. The books are double-entry and real — a duplicate bill
overstates expenses and accounts payable until someone voids it. Prevent it
up front.

## Before every `receive_bill`

1. `get_bills(party_id)` for the vendor and check whether this invoice is already
   booked. Match on the **invoice/receipt number** (kept in the bill `memo`) first;
   fall back to (amount + billing period + due date) when there's no number.
2. If it's already there → stop. Don't re-record. If it's there but wrong, prefer
   `void_bill` + re-create over stacking a second bill.
3. Only if there's no match do you create the new bill.

## Idempotency you can lean on

- `receive_bill` is idempotent on its external ref — re-sending the *same* bill
  returns the existing one rather than creating a duplicate. Use a stable ref per
  source invoice (e.g. the vendor invoice number) so retries are safe.
- `approve_bill` is idempotent: calling it again on an approved bill is a no-op,
  not a second posting.
- `pay_bill` is idempotent on `(bill, idempotency_key)` — reuse the same key for a
  given payment so a retry never double-pays.

## Recurring charges

A monthly subscription produces a *new* invoice each period — that's a new bill,
not a duplicate. Reuse the **same party, contract, and obligation**; only the bill
(and its period/number) is new. Don't create a fresh contract or obligation for
each month — that fragments the spine and breaks the run-rate and unit-economics
views.

## One source, one bill

Each receipt/invoice maps to exactly one bill. If one email bundles several
products, that's one bill with multiple **lines** (each tied to its obligation),
not several bills.
