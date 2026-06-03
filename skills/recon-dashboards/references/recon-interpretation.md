# Reconciliation interpretation

How to read NetCash reconciliation data correctly — what each field means, what counts as a real variance, and what `Balance Changed` status actually signals. Used by the `recon-dashboards` skill, by the `/investigate-variance` command, and by the `troubleshooting` skill when diagnosing reported recon issues.

The dominant failure mode this reference exists to prevent: an agent reading `balance_difference != 0` on a recon record and chasing it as if it were a current variance — when in reality `balance_difference` is normally non-zero on any active account with checks-in-flight or deposits-in-transit, and the actual variance signal lives elsewhere.

## The four-category framework

Each line on a NetCash recon falls into one of four categories, based on which period the GL hit and which period the bank hit:

| Category | GL period | Bank period | What it represents |
|---|---|---|---|
| **Inflow / Outflow Matched** | This period | This period | Cleanly matched activity — no recon work needed |
| **Cleared** | Prior period | This period | GL booked previously, bank cleared this period — closing prior-period outstanding |
| **Outstanding (NetSuite Items)** | This period | Future period | GL booked this period, bank hasn't cleared yet — outstanding checks, deposits in transit |
| **Open Bank Activity** | (not yet) | This period | Bank hit this period with no GL counterpart — the actual variance candidates |

Outstanding NetSuite items are **normal** — every active account with check or DIT activity has them. Open Bank Activity items are the recon work — they represent bank movements that don't yet tie to GL.

## Recon record fields and what they actually mean

The fields on the recon record returned by `getNetCashReconciliations(columnSet='financial')`:

| Field (MCP alias) | Source field | What it represents |
|---|---|---|
| `statement_balance` | `custrecord_ngob_recon_bank_statement_bal` | The bank statement ending balance the user entered |
| `bank_balance` | `custrecord_ngob_recon_bank_balance` | NetCash's computed bank balance from bank activity |
| `gl_balance` | `custrecord_ngob_recon_gl_balance` | NetCash's computed GL balance from GL activity |
| `balance_difference` | `custrecord_ngob_recon_balance_diff` | `gl_balance − bank_balance` — the **gap between GL and Bank balances**, signed GL minus Bank. **NOT a variance.** Non-zero is normal whenever outstanding items exist. |
| `outstanding_bank_balance` | `custrecord_ngob_recon_out_bank_bal` | Net of Open Bank Activity items (bank-side unmatched) |
| `outstanding_gl_balance` | `custrecord_ngob_recon_out_gl_bal` | Net of Outstanding NetSuite items (GL-side outstanding, not yet on bank) |

The key thing to understand: **`balance_difference` is a tautological subtraction (GL − Bank). It's non-zero whenever the two balances differ for any reason, which includes normal outstanding items in flight.** Treating non-zero `balance_difference` as evidence of a recon variance is a documented failure mode (see the worked example below).

## How to assess whether a recon has a real variance

Two signals, in order of authority:

1. **The recon's `status` field.** This is what the user themselves has signed off on.
   - `COMPLETED` / `AUTO_RECONCILED`: user signed off — recon is balanced as far as the user is concerned. Do not surface a variance.
   - `BALANCE_CHANGED`: transactions changed since submission (see below). Surface as "needs re-review" — don't claim a current variance until you've checked.
   - `IN_PROGRESS`, `WORKING_ON_IT`, `STUCK`, `SUBMITTED_FOR_REVIEW`, `NOT_STARTED`: ongoing work — surface progress, don't try to diagnose a variance the user hasn't claimed exists.

2. **Open Bank Activity items.** Bank-side transactions in the period without a GL counterpart. These are the actual variance candidates. To find them via MCP, pull `getNetCashBankActivity` with `isOnlyUnmatched: true` filtered to the recon's account and period. Anything returned is potentially the variance the user is asking about. If nothing is returned, there's no current bank-side variance regardless of what `balance_difference` says.

**Do not** infer a variance from `balance_difference != 0` alone. Many balanced recons have non-zero `balance_difference` because of legitimate outstanding items.

## Balance Changed status — what it actually means

`Balance Changed` (NetCash recon status ID `4`) is set when a transaction (bank or GL) in that account+period is created, modified, or deleted **after the recon was already in `COMPLETED`, `SUBMITTED_FOR_REVIEW`, or `AUTO_RECONCILED`** status. It's a flag saying:

> *The underlying transactions changed since you finished this recon. Reopen and re-review.*

It does **not** automatically mean a variance currently exists. The user may reopen and find:
- The changes are valid and the recon is still balanced → just resubmit
- The changes introduced a real variance → investigate and fix
- The changes were errors → remediate the transactions

The recon record's balance fields (`bank_balance`, `gl_balance`, `balance_difference`, etc.) on a `Balance Changed` recon **reflect the current state** after the transaction changes — they show what the balances are right now, not a snapshot from submission time. A Balance Changed recon with populated balance fields is normal; the values are current.

When the agent sees `status: Balance Changed`:
1. **Don't** treat `balance_difference` as evidence of a current variance.
2. **Do** explain to the user that transactions changed since their submission, and offer to identify which ones (query bank/GL transactions in the period with `lastModifiedDate > recon's submission date`).
3. **Do** check Open Bank Activity for the account+period. If empty, the recon's underlying numbers may still tie — the user just needs to reopen and acknowledge the changes.

## What the cash-position view should and shouldn't flag

The `recon-dashboards` cash-position view should flag accounts that need attention. The correct flagging signal is:

- **Flag** accounts where `status` is `BALANCE_CHANGED` (re-review needed), `STUCK` (stalled), or `NOT_STARTED` for the current period (work hasn't begun).
- **Flag** accounts with Open Bank Activity items in the current period (real bank-side variance candidates).
- **Don't flag** accounts solely because `balance_difference != 0`. That's the GL/Bank gap, which is normal whenever outstanding items exist.

The cash-position view now reads its balances — Account Balance, GL Balance, and `variance` — from `getNetCashAccountBalances` (live, as of date), and joins `status` from `getNetCashReconciliations`. That tool's `variance` is the same GL − Account gap as `balance_difference` here: non-zero is normal, and it is **not** the attention signal — status is.

## Worked example

A recon record with this shape illustrates the failure mode this framework prevents:

- `status: Balance Changed`
- `bank_balance: $2,158,427.93`
- `gl_balance: $2,155,062.18`
- `balance_difference: -$3,365.75`
- `outstanding_gl_balance: -$3,365.75`
- `outstanding_bank_balance: $0.00`

The wrong path: treat `balance_difference: -$3,365.75` as a current variance and chase it through GL amendments, identifying recently-amended journal entries as the cause.

The actual state: Open Bank Activity is empty. The −$3,365.75 is exactly the outstanding GL items the user already has in flight (e.g., two outstanding customer refund checks — GL booked, bank hasn't cleared). The recon is balanced in the dashboard's *"Adjusted Bank Balance == GL Balance"* sense (Difference: 0.00). The Balance Changed status is a flag from earlier transaction changes that the user needs to acknowledge — not a variance signal. The only legitimate finding to surface is if any individual outstanding check is months old and worth a stale-check follow-up.

Correct interpretation:
1. Read `status: Balance Changed` → not a variance signal; reopen-and-review signal.
2. Check Open Bank Activity → empty → no current bank-side variance.
3. Recognize `balance_difference = outstanding_gl_balance` → just the outstanding-items net total, not a variance.
4. Surface to user: *"The −$3,365.75 is two outstanding customer refund checks (not a variance). Status is Balance Changed because transactions changed since your submission — reopen the recon in NetCash to acknowledge them, then resubmit. One of the outstanding checks is 6 months old and may be worth following up on as a stale-check."*
