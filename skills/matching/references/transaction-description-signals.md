# Transaction Description Signals

Bank transaction descriptions contain matching signals that the merchant field alone doesn't capture. Active reading of `custrecord_ngob_banktran_description` is often the difference between a successful match and a transaction stuck in unmatched.

The current entity-mapping system in NetCash searches the merchant field only — not the description (see `../../troubleshooting/references/known-failure-modes.md` for the entity-mapping merchant-only-search limitation). For B2B ACH and wire transfers, the customer/vendor name is usually only in the description, never the merchant field. Reading the description manually is what bridges that gap during a matching pass.

## Signal table

| Signal | Example | Action |
|---|---|---|
| **Invoice numbers** | `INV-1234`, `INVOICE 1042`, `IC-2025-0093` | Match to that specific invoice. The strongest single description signal. |
| **PO numbers** | `PO 47892`, `PURCHASE ORDER 12345` | Match to GL transactions referencing the same PO; verify amount and entity before submitting. |
| **Check numbers** | `CHECK 4521`, `CHK#4521`, `CK 4521` | Match to GL bill payment with the same check number. |
| **Fee deductions** | `$25 FEE DEDUCT`, `LESS $50 WIRE CHG`, `FEE: 25.00` | Calculate net: `invoice - fee = expected bank amount`. Surface the fee in the variance explanation. |
| **Payment IDs** | `PAYMENT ID 2712`, `PAYOUT ID po_abc123`, `STRIPE PAYOUT 2712` | Group all bank lines with the same payment ID; sum and match against an invoice (or invoices) for the relevant entity. |
| **Entity names** | `WIRE FROM ACME CORP`, `ACH PMT OMEGA SERVICES` | Match to that entity's open invoices/bills. Even partial entity strings are usable if the amount is exact. |
| **Count hints** | `COVERING 4 INVOICES`, `BATCH OF 7`, `MULTI-PAY` | Look for N items from the same customer that sum to the deposit. Common with batch payments. |
| **Memo references** | `RE: PROJECT JONES`, `MEMO: MARCH RETAINER` | Cross-reference to invoices or bills with matching memo / job / project fields in NetSuite. |
| **Statement / period markers** | `MARCH 2025 STATEMENT`, `Q1 RETAINER` | Match to a recurring bill / invoice for that period; suggest an automation rule if the pattern recurs. |
| **Reversal markers** | `REVERSAL`, `RETURN`, `NSF`, `CHARGEBACK` | Treat as a reversal — match to the original transaction's reversal pair, not as a new payment. Surface the reversal context. |

## Reading strategy

When you pull unmatched bank transactions, scan the descriptions in three passes:

### Pass 1 — strong references first

Look for invoice numbers, PO numbers, check numbers, payment IDs. These are unambiguous signals. If you find an invoice number, match to that invoice directly — don't waste cycles on amount-only matching when you have a reference.

Strong references can override moderate amount or date variance. A 3-day lag with an exact invoice number reference is a higher-confidence match than a same-day exact-amount match with no reference.

### Pass 2 — fee language

Look for `FEE`, `CHG`, `DEDUCT`, `LESS`, `MINUS`, `NET OF`. When fee language is present and the amount variance fits the stated fee, the variance explanation is in your hand — use it. When fee language is present but the variance doesn't fit, that's a yellow flag; verify before assuming the fee is what you think it is.

### Pass 3 — entity names

For transactions with no reference and no fee language, look for entity names in the description. Bank descriptions for ACH and wire transactions often include the originator/beneficiary name — `ACH PMT ACME CORP`, `WIRE FROM BETA INDUSTRIES`. Cross-reference these against:

1. NetCash entity mappings (`findNetCashEntityMappingsByAlias`)
2. NetSuite customer / vendor lists (`getNetSuiteEntities` if needed)
3. Open invoices / bills for the matched entity

If the entity has only one open item that matches the bank amount, propose the match. If multiple open items match, surface all and ask the user to choose.

### Pass 4 — count hints and batch signals

Phrases like "COVERING 4 INVOICES", "BATCH OF 7", "MULTI-PAY" are explicit hints that you're looking at a one-to-many cash application. Look for N invoices from the same customer that sum to the deposit. Same-day deposits from the same originator are a strong signal even without an explicit count hint.

### Pass 5 — period and memo references

For recurring transactions (monthly retainers, quarterly subscriptions, period-end statements), the description often includes the period being paid. Match against the corresponding period invoice. If the pattern looks like it recurs, flag it for the `automation-rules` skill — the user can save a rule that auto-matches the next occurrence.

## Common description patterns by transaction type

### ACH credits (deposits)

- `ACH CREDIT [originator name] [optional reference]`
- `ACH PMT [customer name] [invoice ref]`
- `WEB CR [merchant] [transaction ID]`

The originator name is the strongest signal. Map it to a customer; check that customer's open invoices.

### Wire transfers

- `WIRE FROM [originator] [optional reference]`
- `INTL WIRE [country] [originator]`
- `FED WIRE [confirmation] [reference]`

Wires are usually high-value and worth extra scrutiny. Always check for fee language ("FEE DEDUCT $25") because international wires nearly always include intermediary deductions.

### Card / processor deposits

- `STRIPE PAYMENT [invoice ref or payout ID]`
- `PAYPAL TRANSFER [reference]`
- `SQUARE INC [date]`
- `MERCHANT SVCS [batch ID]`

Card processor deposits are nearly always net of processor fees (see `references/processor-fees.md`). When the amount is ~3% less than an open invoice for a customer who pays via that processor, the match is high-confidence with a fee variance explanation.

### Checks

- `CHECK 4521`, `CHK 4521`, `CK#4521 [optional payee]`
- For deposited checks: `MOBILE DEPOSIT`, `ATM DEPOSIT`, `BRANCH DEPOSIT`

Check numbers are the strongest possible reference signal — exact matches almost always succeed.

### Bank-internal transactions

- `MONTHLY SERVICE FEE`
- `WIRE FEE`, `INTL FEE`
- `INTEREST EARNED`, `INTEREST PAID`
- `OVERDRAFT FEE`, `NSF FEE`

These don't match to invoices or bills — they need a journal entry. Don't propose a match; instead, suggest the user create an automation rule to auto-create a journal entry for the recurring fee (route to the `automation-rules` skill).

## Anti-pattern: ignoring the description

A common matching failure is to score on amount + date + entity-from-merchant only and skip the description entirely. The description often contains the deciding signal — the invoice number, the fee statement, the entity name that the merchant field doesn't have. Read it.

When you score a Low-confidence amount-only match without checking the description, you risk:

- Missing a strong reference signal that would lift the match to High
- Missing a fee language that explains the variance
- Missing the count hint that turns a 1:1 fail into a 1:N success

Active description reading is part of every matching pass.
