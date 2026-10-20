# Bring in existing books (historical imports)

Use this when the founder wants their history in Economico: the books they kept before this
business was created here (last year's customers, contracts, invoices, receipts, payments).
History is free: anything you record dated before the business was created comes in as an
**import**. It does not count toward the monthly events, and it uses the business's one-time
import allowance instead (1,000 events on Solo, 10,000 on Startup once its first fee is paid,
cumulative). There is nothing to start or finish, and anyone in the company, or their agent, can
record it.

## The procedure

1. **Read the budget.** `business {action: "imports"}` returns `allowance`, `imported`,
   `remaining` and `cutoff`: the instant the business was created on Economico. History dated
   before it is an import.
2. **Size it, before writing anything.** Estimate how many events the history needs from the
   source (records per period). If it will not fit in `remaining`, tell the founder which periods
   fit: history past the allowance is recorded as ordinary events, which count.
3. **Record history at its own dates.** Send each command as you normally would, with
   `effective_at` (and the dates in its input) when the fact happened, before `cutoff`. Every
   event a command appends must be dated before the cutoff, or the whole command counts. Import
   oldest first, a whole period at a time.
4. **Check the receipt.** Each imported receipt says `imported: true`; one without it was
   counted. A retry with the same idempotency key returns the first receipt, so date a command
   correctly before sending it.

Contracts that started before the business was created, and their past invoices and payments,
are history too: record them in [first-session](first-session.md) at their actual dates.

Economico's daily schedule never records history for you. For a contract dated before the
business was created (the UTC date of `cutoff`), it records only what falls due after you record
the contract: every earlier invoice, service period and expiry is yours to bring in from the old
books, however long after signup you import it, and nothing you import is invoiced again. An
invoice is the schedule's when it falls due after you recorded the contract: its period's start
(billed in advance) or end (billed at its end), plus the schedule's `offsetDays`, and never
before the contract's acceptance. `activities plan` still lists the earlier periods
for any range you ask. A contract dated on or after the day the business was created is current:
the schedule records its periods from that day, including ones it missed. Record a contract from the old books
and its acceptance in the same session: the schedule starts when the contract is first recorded,
so a proposal recorded days before its backdated acceptance would have those days' periods
recorded by the schedule.

Economico's own invoices for the plan arrive in the books by themselves: never record or import
them, even from an emailed PDF (recording one is refused).

## What is never an import

Decisions made today, whatever date they carry: membership and ownership changes, share links,
the business's own profile and settings (`business.*`), accounting-policy decisions (fiscal
year, framework, currency, legal identity), messages and connections, ingestions
(`ingestions.*`) and cancellation requests. They are recorded as ordinary changes. Do not re-import history a predecessor business's books already hold.
