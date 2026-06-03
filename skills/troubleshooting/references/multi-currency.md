# Multi-Currency Edge Cases

Cross-currency match guidance for NetCash troubleshooting. Covers currency mismatch, FX rate selection, the `exchangeRate` parameter on `submitNetCashManualCashApplication`, and rounding tolerance.

For the broader FX rounding pattern (a small unmatched delta on an otherwise-aligned cross-currency match — handled structurally by NetCash's `isBookRounding` flag), see `known-failure-modes.md` § FX rounding. This file covers the mechanics and edge cases.

---

## Currency mismatch — three distinct cases

When a user reports "the match won't go through because of currency", confirm which case applies:

### Case 1: Bank account and GL invoice are the same currency, but match fails

This is **not** a multi-currency issue. Re-route to one of the other named failure modes:

- Amount differential matches a PSP fee structure → PSP variance
- Entity is in description but not merchant → entity-mapping merchant-only-search limitation

### Case 2: Bank account and GL invoice are different currencies (true cross-currency match)

This is a cross-currency cash application. NetCash supports it via `submitNetCashManualCashApplication` with the `exchangeRate` parameter. See "exchangeRate parameter" below.

### Case 3: Vendor or customer record doesn't have the needed currency enabled

NetSuite-level error: "Currency not valid for entity" or "Entity not valid for subsidiary in this currency". The fix is on the entity record's Financial / Subsidiary tab — enable the currency, retry the match. Inspect via `ns_runCustomSuiteQL` joined to `customer_subsidiary_map` / `vendor_subsidiary_map` (customers and vendors) plus the entity's currency-enabled list, or `getNetSuiteEntities` for a quick entity lookup.

This is a NetSuite setup issue, not a NetCash defect.

---

## FX rate selection — which rate, which date

NetSuite supports several FX rate types (Average, Spot, Historical, Custom). The rate that should drive the cash application depends on the customer's FX policy.

When the user is matching a foreign-currency payment to an invoice:

1. **Default behavior** — `submitNetCashManualCashApplication` without an explicit `exchangeRate` parameter uses NetSuite's default consolidation rate for the transaction date. This works for the majority of customers who use Average rate or daily Spot rate.

2. **Explicit override** — pass `exchangeRate` when:
   - The customer's FX policy uses a rate type NetSuite isn't applying by default (e.g., transaction-date Spot rate when the default is month-end Average).
   - The user has confirmed the bank transaction's settlement rate (rare — typically only available from the bank's settlement notice, not from the deposit description).
   - Reconciliation requires aligning to a specific historical rate the customer captured at invoice time.

3. **Reading the rate that was applied** — to inspect what rate NetSuite used on an existing match, pull the underlying transaction via `ns_runCustomSuiteQL` against `transaction` joined to `transactionAccountingLine` and read the FX rate field. The implied rate from a NetCash bank transaction is `functional_amount / transactional_amount`.

---

## The `exchangeRate` parameter on `submitNetCashManualCashApplication`

The parameter is optional. Pass it when the default consolidation rate doesn't match the customer's FX policy or the user has an explicit rate to apply.

When passing `exchangeRate`:

- The rate is the foreign-currency-to-functional-currency rate (e.g., for a USD subsidiary applying a EUR payment, `exchangeRate = USD per 1 EUR`, like `1.0825`).
- The rate is applied to the GL transaction-currency amount; the bank transaction's functional-currency amount stays as-is.
- If the resulting functional-currency totals don't reconcile within rounding tolerance, the write will fail with an FX-specific error. Surface the error to the user — don't retry with random rate adjustments.

**Default to omitting the parameter** unless the user has surfaced a specific FX policy reason. Most customers' FX setup is correct out of the box and the explicit override creates more risk than it removes.

---

## Rounding tolerance

The functional-currency total of a cash application has to balance within a small tolerance — typically NetSuite enforces 0.01 in the functional currency. Beyond that, the write fails.

For a cross-currency match, the rounding tolerance compounds:

- One line: 0.01 functional currency
- Multi-line cash application (e.g., one payment applied to three invoices): up to 0.03 functional currency cumulative rounding

If the discrepancy exceeds the cumulative tolerance:

1. First check whether it's actually FX rounding, not a rate-selection issue. The discrepancy from rate selection is usually larger than from rounding (often > 0.5% of the transaction). If the discrepancy is < 0.1% of the transaction, it's almost certainly rounding.
2. If rounding, post the residual to the FX gain/loss account configured on the bank account or NetCash sourcing record. NetCash's variance handling on `submitNetCashManualCashApplication` covers this when the variance amount is supplied.
3. If rate-selection, see "FX rate selection" above — pass `exchangeRate` with the correct rate.

---

## Cross-subsidiary cross-currency — the compound case

When the customer is on OneWorld and the cash application crosses subsidiaries AND currencies, both constraints apply:

- The entity must have the bank-account subsidiary enabled (NetSuite-level Subsidiary tab on the entity record).
- The entity must have the bank-account currency enabled (NetSuite-level Financial tab on the entity record).
- The cash application uses the source-subsidiary's functional currency for the FX rate.

This case has historically generated the most support tickets. Walk through both checks (subsidiary enablement, then currency enablement) before assuming an FX defect. Most failures resolve to one of these two NetSuite-level setup issues.

---

## Diagnostic checklist for a reported FX issue

1. Confirm the bank account currency and the GL invoice / bill currency. If same currency, re-route per Case 1 above.
2. Pull the entity record via `ns_runCustomSuiteQL` against `customer` / `vendor` joined to the currency-enabled list, or `getNetSuiteEntities` for a quick entity lookup, and confirm the needed currency is enabled on the Financial tab.
3. If OneWorld, also confirm the entity has the bank-account subsidiary enabled on the Subsidiary tab.
4. Compute the implied FX rate from existing matches and compare to the customer's expected rate. If mismatched by > 0.5%, this is likely rate selection, not rounding.
5. If rounding (< 0.1% off), match with variance posting the residual to the FX gain/loss account.
6. If rate selection, retry the match with explicit `exchangeRate` parameter set to the customer's policy rate.
7. If all checks pass and the match still fails, escalate to Netgain Support per `escalation-criteria.md` — include the bank account ID, GL invoice ID, the rate the user expected, and the rate NetCash applied.
