# Known patterns — recurring rule shapes the `automation-rules` skill should recognize

Reference for the `automation-rules` skill. These are the patterns that recur across NetCash customers. When you see one of these signatures in matched data, you have a strong starting point — but always verify field values, run the pre-creation checklist, and call `validateNetCashAutomationRuleFields` before creating.

Each pattern below shows the signature, the recommended rule type, the typical filter shape (with field IDs as placeholders — pull the live IDs from `getNetCashAutomationRuleFieldOptions`), the expected match count, and a sample dialogue showing how to present it.

## 1. Recurring vendor — check-number GL Match

**Signature:** Same vendor merchant, withdrawals matched to Bill Payments, check numbers consistently match `transaction.otherrefnum`. Exact amount, date gap typically 1-3 days.

**Rule type:** Type 1 — GL Match.

**Typical filter + match-criteria shape:**

- Filter: bank merchant CONTAINS "VENDOR NAME"
- Match: bank check number CONTAINS GL `transaction.otherrefnum`
- `amountVariance: 0`, `dateTolerance: 5`

**Expected match count:** 5+ for high confidence; 3-4 for medium.

**Sample dialogue:**

> **ACME Corp — Check Number Match (GL Match)**
> - 12 matches over 90 days (high confidence)
> - All withdrawals matched to Bill Payments
> - Bank merchant always "ACME CORP" — verified in merchant field
> - Check number consistently matches GL otherrefnum
> - Exact amount match every time, date gap averages 2 days
> - No existing rule covers this pattern (active or inactive)
>
> Proposed rule:
> - Filter: bank merchant CONTAINS "ACME CORP"
> - Match: bank check number CONTAINS GL otherrefnum
> - Amount variance: 0 | Date tolerance: 3 days
> - Group: Checking Accounts | Status: Testing | Require review: on
>
> Want me to create this rule?

---

## 2. Cash Application date defaults — `dateField` and `dateTolerance`

Before drafting any Type 4 rule that applies bank activity to open AR invoices or AP bills, set these defaults unless the matched-pair data argues otherwise:

- **`dateField: true`** — anchors the tolerance window on the invoice/bill **due date** rather than the issue date. For Net 15/30/60 terms, the bank deposit (AR) or withdrawal (AP) lands near the due date, not the tran date. Anchoring on tran date forces the tolerance to span the full payment-term window plus normal early/late variance — almost always wrong. (Help text in the rule builder: *"When enabled, date tolerance is based on the invoice/bill due date. If no due date exists, the transaction date is used instead."*)
- **`dateTolerance: 30`** — even when anchored on due date, payments arrive early, on time, and late. On AR, customers pay across a 30-day spread around due date. On AP, you control the payment date but batch runs, discount captures, cash-flow timing, and check-clear lags produce a similar spread around bill due date. 30 days covers the realistic window for both directions. Tighter (5–10) regularly drops legitimate matches; wider (60+) starts pulling in stale items that should be flagged, not auto-applied.

**Exception — PSP deposits (Stripe, PayPal, Square — Section 3 below):** PSP payouts settle in days, not weeks, and are anchored on the payout date rather than customer due dates. Keep `dateTolerance` at 3–5 and leave `dateField` at default (tran date) for PSP rules.

When proposing a Type 4 rule, surface the date-anchor choice explicitly in the proposal — don't bury it in numerical settings. Example: *"Date anchor: invoice due date | Tolerance: 30 days (covers Net-30 terms plus early/late variance)"*.

---

## 3. PSP fees — Stripe / PayPal / Square Cash Application

**Signature:** Recurring deposits from a payment processor with the bank credit consistently 2-3% lower than the GL invoice amount. Multiple invoices grouped under one deposit.

**Rule type:** Type 4 — Cash Application with `isAmountPercentage: true`.

**Typical filter + match-criteria shape:**

- Filter: bank merchant CONTAINS "STRIPE" (or "PAYPAL", "SQUARE")
- Filter: bank credit GREATER THAN 0 (scope to deposits)
- Match: bank description CONTAINS GL `transaction.memo` (Stripe payout ID often appears in both)
- `amountVariance: 4`, `isAmountPercentage: true`, `dateTolerance: 5`
- `groupingCriteria`: `{ "source": 2, "field": "transaction.memo" }` if multiple invoices share the Stripe payout ID; otherwise omit

**Expected match count:** typically 8+ deposits over 90 days for active e-commerce customers.

**Fee references** — use to size `amountVariance`:
- Stripe — 2.9% + $0.30 per transaction.
- PayPal — ~2.9% + $0.30 per transaction.
- Square — 2.6% + $0.10 per transaction.

**Sample dialogue:**

> **Stripe Cash Application (Cash Application)**
> - 18 matches over 90 days (high confidence)
> - All deposits matched to one or more open invoices
> - Bank merchant always contains "STRIPE", credit consistently 2.9% below sum of matched invoice amounts
> - Stripe payout ID appears in both bank description and GL memo on every match
> - 6 of 18 deposits cover multiple invoices for the same payout ID — grouping warranted
>
> Proposed rule:
> - Filter: bank merchant CONTAINS "STRIPE" AND bank credit > 0
> - Match: bank description CONTAINS GL transaction.memo
> - Group GL by transaction.memo
> - Amount variance: 4 (percentage) | Date tolerance: 5 days
> - Group: Operating Accounts | Status: Testing | Require review: on
>
> Want me to create this rule?

---

## 4. Bank-to-bank transfers

**Signature:** Outflow on one account paired with inflow on another, same amount, dates within 0-2 days, descriptions contain transfer keywords.

**Rule type:** Type 3 — Create Transfer.

**Typical filter shape:**

- Filter: bank description CONTAINS "TRANSFER" (or "TRNF", "XFER", "TXFR"), `flow: 3`
- `isCopyIntercompanyValues: false` unless intercompany pattern is observed.

**Expected match count:** 3+ for monthly transfers; weekly transfers typically 8-12 over 90 days.

**Sample dialogue:**

> **Operating to Reserve Transfer (Create Transfer)**
> - 9 matches over 90 days (high confidence)
> - Outflow on operating always paired with inflow on reserve same day
> - Description contains "TRANSFER" on both sides — verified
> - No existing rule
>
> Proposed rule:
> - Filter: bank description CONTAINS "TRANSFER" (both directions)
> - Group: All Accounts | Status: Testing | Require review: on
>
> Want me to create this rule?

**Important:** verify both sides exist (outflow AND inflow) before proposing. A one-sided "transfer" pattern is usually a vendor wire (Type 1 or Type 2), not a real transfer.

---

## 5. Money-market fund dividend — Create Transaction

**Signature:** Monthly cadence deposits with description containing the fund's name + "Dividend" or similar income keyword. No pre-existing GL transaction — user manually creates JEs to recognize the income. Variable amount.

**Rule type:** Type 2 — Create Transaction (Journal Entry).

**Typical filter + template shape:**

- Filter: bank description CONTAINS "MMF" AND CONTAINS "Dividend" (two filters with AND logic; substitute the actual fund-name keyword that appears in the bank descriptor)
- `transactionType: 1` (Journal Entry)
- Template: `[{ "source": 1, "memo": "Money Market Fund Dividend Income" }]` — minimal, GL account set in NetCash before activation.

**Expected match count:** 3-6 monthly observations over 90 days.

**Sample dialogue:**

> **Money Market Fund Dividend Income (Create Transaction)**
> - 6 matches over 90 days, all deposits, monthly cadence
> - Description always contains "MMF" and "Dividend" — verified in description field
> - No matching GL transaction exists; user created JEs manually each month
> - No existing rule
>
> Proposed rule:
> - Filter: bank description CONTAINS "MMF" AND CONTAINS "Dividend"
> - Template: Journal Entry, memo "Money Market Fund Dividend Income"
> - GL account: not set — you'll configure that in NetCash before activating
> - Group: Bank Accounts | Status: Testing | Require review: on
>
> Want me to create it?

This is the canonical example for "no existing GL → Create Transaction" patterns. The same shape applies to any income that hits the bank without a matching invoice or deposit record.

---

## 6. ATM withdrawals — Create Transaction

**Signature:** Withdrawals with bank merchant or description containing "ATM" or "CASH WITHDRAWAL". Usually no pre-existing GL.

**Rule type:** Type 2 — Create Transaction.

**Typical filter + template shape:**

- Filter: bank description CONTAINS "ATM" OR `(source: 1, field: <bank merchant>, operator: 4 (ANY OF), value: "ATM,CASH WITHDRAWAL")`
- Template: `[{ "source": 1, "memo": "ATM withdrawal" }]`

**Expected match count:** depends on customer; 3+ over the lookback is the floor.

**Sample dialogue:**

> **ATM Withdrawals (Create Transaction)**
> - 5 matches over 90 days
> - All withdrawals, description contains "ATM" — verified
> - No matching GL; user created JEs manually
>
> Proposed rule:
> - Filter: bank description CONTAINS "ATM"
> - Template: Journal Entry, memo "ATM withdrawal"
> - GL account: not set — configure in NetCash before activating
> - Group: Bank Accounts | Status: Testing | Require review: on
>
> Want me to create it?

---

## 7. Monthly bank service fees — Create Transaction

**Signature:** Small recurring withdrawals with description containing "service charge", "monthly maintenance", "fee", "FDIC", or similar. No pre-existing GL.

**Rule type:** Type 2 — Create Transaction.

**Typical filter + template shape:**

- Filter: bank description CONTAINS "service charge" OR "monthly maintenance" OR "fee" — usually a single keyword that's specific to the bank.
- Template: `[{ "source": 1, "memo": "Bank service fee" }]`

**Expected match count:** 3+ monthly observations over the lookback.

**Sample dialogue:**

> **Monthly Bank Service Fee (Create Transaction)**
> - 3 matches over 90 days, monthly cadence, exact $25.00 each
> - Description always "MONTHLY SERVICE CHARGE" — verified
> - No matching GL; user created JEs manually
>
> Proposed rule:
> - Filter: bank description CONTAINS "MONTHLY SERVICE CHARGE"
> - Template: Journal Entry, memo "Monthly bank service fee"
> - GL account: not set — configure in NetCash before activating
> - Group: Operating Accounts | Status: Testing | Require review: on
>
> Want me to create it?

This pattern often produces high-confidence proposals because the cadence is exact and the description is stable.

---

## 8. Coverage check — explaining why a transaction is or isn't covered by an existing rule

When the user asks "do we already have a rule for X?", or proposes adding one, run the coverage check before suggesting a new rule.

**Procedure:**

1. Pull the existing rule's complete filter set.
2. Pull the candidate transaction's actual field values.
3. Test each filter individually, then test the AND of all filters together. A transaction is covered only if it matches every filter.
4. Report each filter and whether the transaction matches it.

**Sample dialogue (covered):**

> You have "Travel Expenses" with two filters:
> - Filter 1: Offset GL IS "6300 Travel Expenses"
> - Filter 2: Description NONE OF empty
>
> Your 4 Uber matches have GL = 6300 and non-empty descriptions — they match both filters. Already covered.

**Sample dialogue (not covered, leading into a Type 2 proposal):**

> Those aren't covered — their Offset GL is null, so they fail Filter 1 of the Travel Expenses rule.
>
> I see 6 matches, all deposits, monthly cadence. Description always contains "MMF" and "Dividend" — verified in the description field. No existing GL to match against — the user created JEs manually to reconcile these.
>
> [proceeds with the money-market dividend proposal]

The structure — name the existing rule, list the filters, explain which one fails — is what makes the coverage check legible. Don't say "it's not covered" without showing why.

---

## 9. Rules to flag as unusual when proposing

These shapes are technically valid but rarely correct. If the matched data points toward one of these, flag it as unusual in the proposal and ask for explicit user justification before creating.

- **Type 1 with only filters and no `matchCriteria`** — narrows the candidate pool but cannot identify the correct GL transaction. Almost always missing match criteria.
- **Type 1 or Type 4 with `isRequireAll: false`** — OR logic across match criteria. Real OR cases exist (e.g., either check number OR otherrefnum can identify the GL) but they're rare.
- **Type 1 with `isMatchNulls: true`** — pairs null bank values with null GL values. Only valid when the matchgroup data shows null-to-null pairings explicitly.
- **Type 3 with intercompany flags enabled** — only when the matched pattern shows intercompany values being copied across.
- **Wide `amountVariance` (> 5) without percentage flag** — usually an indicator that the rule is trying to absorb a different problem (e.g., FX rounding, wrong rule type).

When you flag one of these, name what's unusual and ask the user to confirm or adjust.
