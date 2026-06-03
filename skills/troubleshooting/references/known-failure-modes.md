# Known Patterns and Limitations

Five named patterns and limitations the NetCash troubleshooting skill recognizes by name. The first two are **expected variance patterns** that NetCash already handles via dedicated features (`isCreateDifferenceJournal`, `isBookRounding`); the remaining three are real limitations or setup issues. For each: symptom, diagnosis steps, remediation, and when to escalate. Together these cover the majority of NetCash troubleshooting tickets.

If a symptom doesn't map to any of these five, fall back to the issue-type-based diagnostic in the SKILL.md Workflow section, step 3.

> Frame these accurately when talking to the user. A controller hearing "FX rounding is a known failure mode" assumes a defect; hearing "FX rounding is an expected pattern, and NetCash has a built-in setting for it" reaches for the right control. Use the second framing.

---

## 1. PSP variance (Stripe / PayPal / Square fee deductions) — expected pattern, structured handling

### Symptom
A bank deposit lands ~3% smaller than the GL invoice it should be matching. The user sees an "amount mismatch" or the auto-matcher doesn't pair the deposit to the invoice.

Common signature: bank description contains `STRIPE`, `PAYPAL`, `SQUARE`, `SQ *`, or a PSP-specific transaction ID. The deposit amount is consistently less than a clean GL amount, with the differential matching a percentage-plus-fixed-fee structure.

### Diagnosis steps

1. Pull the bank transaction with full detail: `getNetCashBankActivity` with `columnSet: detailed`, filtered to the affected deposit. Check the description and merchant fields for the PSP signature.
2. Pull the candidate GL invoices: `getNetCashGLActivity` or `getNetCashCashApplicationTransactions` for open invoices in the same currency, around the same date.
3. Calculate the expected net for the candidate invoice using the PSP's published fee structure:
   - Stripe (US, card): `net = invoice × (1 − 0.029) − 0.30` (2.9% + $0.30)
   - PayPal (US, standard): `net = invoice × (1 − 0.029) − 0.30` (~2.9% + $0.30; varies by account tier)
   - Square (US, in-person): `net = invoice × (1 − 0.026) − 0.10` (2.6% + $0.10)
   - Square (US, online keyed): `net = invoice × (1 − 0.029) − 0.30`
   - ACH via Stripe: typically 0.8% capped at $5
4. If the bank deposit equals the calculated net (within a penny or two for rounding), this is PSP variance.

### Remediation

Use NetCash's structured handling: apply the bank deposit to the GL invoice using `submitNetCashManualCashApplication` with `isCreateDifferenceJournal: true` and `offsetAccount` set to the customer's PSP fee expense account (typically "Stripe Fees", "PayPal Fees", "Merchant Processing Fees", or similar). The submit tool posts the differential to the offset account automatically — this is a first-class NetCash feature, not a workaround.

Surface the calculated fee in the proposed write so the user can confirm the math before the write goes through.

If this is a recurring pattern (the same PSP, the same fee structure, ≥ 3 times observed), recommend the user invoke `/suggest-rules` to build a Cash Application rule with PSP fee handling — this is the canonical use case for the `automation-rules` skill's PSP fee pattern.

### When to escalate

Don't escalate. PSP variance is an expected reconciliation pattern that NetCash handles by design — not a defect.

---

## 2. FX rounding (multi-currency rounding differences) — expected pattern, structured handling

### Symptom
A cross-currency match fails or the matched amount disagrees with the GL by a small amount (typically < 1% of the transaction). The user is reconciling a foreign-currency bank account or applying a payment in one currency to an invoice in another.

### Diagnosis steps

1. Confirm the transaction is cross-currency: `getNetCashBankActivity` for the bank-side currency, `getNetCashCashApplicationTransactions` for the GL-side invoice currency.
2. Identify the FX rate NetSuite used: the bank transaction stores its functional-currency amount; the GL stores its transactional-currency amount. The implied FX rate = functional / transactional.
3. Compare to the user's expected rate (from their FX policy — daily, monthly, transaction-date, etc.).
4. Compute the rounding tolerance: the discrepancy is FX rounding if it's less than the smallest currency unit (typically 0.01) per line × number of lines in the match.

For deeper FX guidance, see `multi-currency.md`.

### Remediation

NetCash handles this via the `isBookRounding` flag on Cash Application rules: when set, NetCash posts a small rounding entry to the configured FX gain/loss account automatically rather than rejecting the match. For a one-off manual match, use `submitNetCashManualCashApplication` with `isCreateDifferenceJournal: true` and `offsetAccount` set to the FX gain/loss account configured on the bank account or NetCash sourcing record. This is structured handling built into NetCash — not a workaround.

If the discrepancy exceeds tolerance, surface to the user — this is likely an FX rate selection issue (wrong rate type or wrong date used), not rounding. Use `submitNetCashManualCashApplication` with the explicit `exchangeRate` parameter (see `multi-currency.md`).

### When to escalate

Escalate if NetSuite is rejecting the match write with an FX-specific error after multiple rate-selection attempts.

---

## 3. Single-subsidiary view shape on `getNetCashReportMatches` — view-shape difference

### Symptom
A controller on a single-subsidiary NetSuite instance sees subsidiary-grouped output that's empty, redundant, or just the same single subsidiary repeated across every row. The tool itself returns data — the subsidiary breakdown just isn't meaningful on a single-subsidiary tenant.

This is **not a tool failure**. `getNetCashReportMatches` runs on every NetCash instance regardless of OneWorld status. The issue is purely how skills present the data.

### Diagnosis steps

1. Detect OneWorld per the rule in plugin-level `CLAUDE.md`: call `ns_getSubsidiaries` and ignore consolidated entries (negative IDs) — more than one real subsidiary → OneWorld; zero or one → not OneWorld. Fallback: `ns_runCustomSuiteQL` with `SELECT COUNT(*) AS sub_count FROM subsidiary` (`sub_count > 1` → OneWorld; `sub_count = 1`, or a "table not found" error, → not OneWorld).
2. The plugin-level `CLAUDE.md` `<oneworld_detection>` section is the single source of truth — every skill that renders subsidiary-grouped views must check OneWorld status first to decide the output shape.

### Remediation

Skills must detect OneWorld before rendering subsidiary-grouped output, not before calling the tool. If the customer is non-OneWorld:

- `recon-dashboards` continues to call `getNetCashReportMatches` for match-progress enrichment but drops the subsidiary column. The cash-position view groups by account only.
- `troubleshooting` continues to use `getNetCashReportMatches` for orphan-match and amount-mismatch diagnostics; the diagnostic still works, just without the per-subsidiary breakdown.

Surface to the user when a subsidiary breakdown was dropped because they're on a single-subsidiary instance. Don't silently degrade — say so once at the bottom of the response.

### When to escalate

Don't escalate. This is a view-shape difference, not a defect or a tool limitation.

---

## 4. Authorized Date setup (credit cards: transaction date vs. posting date)

### Symptom
Credit-card transactions match against the wrong period, or fall in or out of a reconciliation when the user expected the opposite. Common when the customer is reconciling a corporate card or AMEX.

The underlying issue is the dual-date problem on credit cards: the authorized date (when the cardholder swiped) can be days or weeks earlier than the posting date (when the card issuer settled to the merchant). NetSuite GL typically uses the authorized/transaction date; bank feeds typically post on the posting date. The reconciliation cutoff has to choose one.

### Diagnosis steps

1. Pull the affected bank transaction with `columnSet: detailed`: `getNetCashBankActivity`. Inspect the two date fields:
   - `custrecord_ngob_banktran_authorized_date` — labeled **Authorized Date** in the NetCash UI (typically the authorized / swipe date)
   - `custrecord_ngob_banktran_date` — the standard transaction date (typically the posting date for credit-card feeds)
2. Determine which date the customer's reconciliation should cut over on. For corporate cards this is almost always the Authorized Date (swipe date) so the reconciliation aligns to the cardholder's behavior, not the issuer's settlement timing.
3. Confirm the bank account's reconciliation configuration uses the Authorized Date as the cutover field.

### Remediation

If the customer's account is configured to use posting date (`custrecord_ngob_banktran_date`) when they should be using Authorized Date (`custrecord_ngob_banktran_authorized_date`), update the bank-account reconciliation configuration. This is a one-time setup change — surface to the user that it will reclassify in-flight transactions and ask for explicit confirmation before the change.

For individual transactions where the dates disagree by more than expected, inspect the underlying credit-card statement to confirm the correct date.

> Naming note: the field's NetSuite-displayed label is "Authorized Date". Some documentation refers to the same concept as "Statement Eligibility Date"; in user-facing output, always use the label NetSuite shows the controller — "Authorized Date".

### When to escalate

Escalate if the bank-account configuration appears correct but transactions are still landing in the wrong period — this could be a credit-card feed mapping defect.

---

## 5. Entity-mapping merchant-only-search limitation

### Symptom
The user has a configured entity alias (e.g., "ACME CORP" → vendor record `Acme Corporation, LLC`). A bank transaction lands that clearly involves Acme — the entity name is in the bank description — but auto-matching does not resolve the entity. The rule that depends on the entity match doesn't fire.

This is most common on B2B ACH and wire transactions, where the bank's `merchant` field is empty and the actual counterparty name only appears in the `description` field.

### Diagnosis steps

1. Pull the affected bank transaction with `columnSet: detailed`: `getNetCashBankActivity`.
2. Check the merchant field (`custrecord_ngob_banktran_merchant`) and the description field (`custrecord_ngob_banktran_description`).
3. If the merchant field is empty and the entity name only appears in the description, this is the merchant-only-search limitation.

`findNetCashEntityMappingsByAlias` (and the underlying `matchEntityToMerchant()` resolution path used by NetCash auto-matching) searches against `custrecord_ngob_banktran_merchant` only — not `custrecord_ngob_banktran_description`. The alias is configured correctly; the match path doesn't see it because the merchant field is empty.

### Remediation

Workarounds the user can apply today:

1. **Build a CONTAINS rule on description.** Instead of relying on the alias resolution, use the `automation-rules` skill to build a GL-Match or Cash-Application rule with operator `CONTAINS` on the description field. This catches the bank-text → entity link directly, without going through the merchant-field alias resolution.
2. **Use the `getNetCashEntityMapping` data alongside a custom rule.** If many entities share this issue, surface a list and recommend the rule pattern.

### When to escalate

Don't escalate as a bug — this is a documented limitation. Do escalate if the user's alias is configured correctly, the merchant field is populated (not empty), and matching still doesn't resolve — that would be a separate issue.
