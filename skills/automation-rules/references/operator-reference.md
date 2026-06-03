# Operator reference — NetCash automation-rule filter and match operators

Reference for the `automation-rules` skill. Operator IDs are validated by `validateNetCashAutomationRuleFields` and surfaced live by `getNetCashAutomationRuleFieldOptions`. Use the live list as the source of truth; this table documents intent and worked examples.

## Operator IDs

| ID | Operator | Use for | Notes |
|---|---|---|---|
| 1 | IS | Exact equality | Use only when the bank data is clean and deterministic (e.g., normalized payment channels, known-stable merchant strings). Prefer CONTAINS for free-text fields. |
| 2 | CONTAINS | Substring match | **Default for text fields** — bank data is noisy, descriptions vary, merchant names get truncated. Use unless you have a specific reason for IS or BEGINS WITH. |
| 3 | BEGINS WITH | Prefix match | Use when a bank prefixes a known token (e.g., "ACH CREDIT *", "WIRE OUT *"). |
| 4 | ANY OF | Match any value in a list | Use for known-finite enumerations (e.g., a small set of payment channels). Value is a comma-separated list. |
| 5 | NONE OF | Exclude values | Use to scope out specific known-bad strings. Most common when paired with another filter to narrow further. |
| 6 | GREATER THAN | Numeric / date threshold | Use for amount thresholds (filter > 0 to scope to credits or debits) and date thresholds. |
| 7 | LESS THAN | Numeric / date threshold | Use for upper-bound amount and date thresholds. |

## Default operator selection rule

For text fields, default to **operator 2 (CONTAINS)** unless you have evidence the data is normalized.

- Bank merchants are inconsistent across sources (Plaid normalizes, direct REST APIs often don't).
- Bank descriptions are free text and frequently contain trailing references, dates, or check numbers that break exact equality.
- BEGINS WITH is useful when a feed prepends a known token and the rest is variable.

## Worked examples

### Example 1 — CONTAINS for noisy merchant data

Bank merchant strings observed for the same vendor across 14 transactions:

- "STRIPE TRANSFER ST-12J9KK"
- "STRIPE TRANSFER ST-99XX"
- "Stripe Inc"
- "STRIPE PAYMENT"

Rule:

```json
{ "source": 1, "field": "<bank merchant field>", "operator": 2, "value": "STRIPE" }
```

Operator 1 (IS) would catch zero of the four. Operator 2 (CONTAINS) catches all four.

### Example 2 — BEGINS WITH for prefixed references

Bank description strings:

- "WIRE OUT REF#WX-44291"
- "WIRE OUT REF#WX-51002"
- "WIRE OUT REF#WX-52119"

Rule:

```json
{ "source": 1, "field": "<bank description field>", "operator": 3, "value": "WIRE OUT" }
```

BEGINS WITH (3) is more precise than CONTAINS (2) here — it scopes to wire outflows specifically and avoids inadvertently catching inbound wires that happen to mention "WIRE OUT" later in their description.

### Example 3 — ANY OF for finite enumerations

Bank payment channel field has values like `ach`, `check`, `wire`, `card`. To scope a rule to electronic-only payments:

```json
{ "source": 1, "field": "<bank payment channel field>", "operator": 4, "value": "ach,wire" }
```

Comma-separated list. NetCash splits on the comma during evaluation.

### Example 4 — NONE OF as a guardrail

Travel Expenses rule with two filters using AND logic:

```json
[
  { "source": 2, "field": "transactionAccountingLine.account", "operator": 1, "value": "<6300 internal ID>" },
  { "source": 1, "field": "<bank description field>", "operator": 5, "value": "" }
]
```

The second filter (NONE OF empty) excludes transactions with a null/empty description. The pair scopes to "GL account is 6300 AND bank description is non-empty" using AND logic.

### Example 5 — GREATER THAN for direction filtering

Cash Application rule scoped to deposits only (customer payments):

```json
{ "source": 1, "field": "<bank credit field>", "operator": 6, "value": "0" }
```

For withdrawals only (vendor payments):

```json
{ "source": 1, "field": "<bank debit field>", "operator": 6, "value": "0" }
```

GREATER THAN (6) applied to the credit or debit field is the canonical pattern for direction-scoping a Cash Application rule.

## Match-criteria operators (Type 1 GL Match, Type 4 Cash Application)

Match criteria pair a bank field to a GL field. The same operator IDs apply, but the most common combinations are:

- Operator 1 (IS) — for deterministic pairings (check number → otherrefnum where both are normalized).
- Operator 2 (CONTAINS) — for fuzzy text pairings (bank merchant → GL entity name where bank data is noisy).

```json
{ "bankField": "<bank check number field>", "operator": 2, "glField": "transaction.otherrefnum", "isMatchNulls": false }
```

`isMatchNulls`: only `true` when null-to-null pairings are explicitly observed in the matchgroup data. Default is `false`.

## Common match-criteria pairings

These are the workhorse pairings for GL Match and Cash Application rules. Use them as starting points; verify field IDs against `getNetCashAutomationRuleFieldOptions` before using.

- **merchant ↔ entity**: bank merchant CONTAINS GL `NVL(transactionLine.entity, transaction.entity)` — typical for vendor payments where the bank shows a merchant name and the GL shows the entity record.
- **merchant ↔ memo**: bank merchant CONTAINS GL `transaction.memo` — when the GL entry's memo field is the source of truth for the vendor.
- **description ↔ memo**: bank description CONTAINS GL `transaction.memo` — common for fee-style transactions.
- **check # ↔ otherrefnum**: bank check number IS or CONTAINS GL `transaction.otherrefnum` — the canonical pattern for matched check payments.
- **check # ↔ tranId**: bank check number IS or CONTAINS GL `transaction.tranId` — when the document number is the natural key.

## When to escalate operator strictness

Start with CONTAINS (2). Tighten to IS (1) only when:

- The bank data is verified normalized for the field across all sample records.
- The user reports false positives from CONTAINS that wouldn't occur under IS.
- The match criteria need exact equality to be valid (rare — usually only for ID-style fields).

If escalating from CONTAINS to IS, re-run the pattern check on the matchgroup data: a stricter operator must still cover all the sample matches.

## Numeric semantics

- `amountVariance: 0` — exact match required. Use for non-fee-bearing transactions.
- `amountVariance: 0.05` to `1.00` — minor rounding tolerance. Use for FX-rounded transactions (very small absolute deltas).
- `amountVariance: 3-5` with `isAmountPercentage: true` — percentage tolerance, used for processor fees in the 2-3% range.

`amountVariance` is always a plain number, never a string. Same for `dateTolerance` (1, 3, 5, etc.).

## Date semantics

- `dateTolerance: 0-1` — same-day. Use for card transactions, electronic payments where the bank and GL post on the same day.
- `dateTolerance: 3-5` — ACH and electronic payments with normal settlement delay.
- `dateTolerance: 5-7` — paper checks. Float varies; 5-7 covers most cases.
- `dateTolerance: 1-3` — wires. Same-day or next-day usually, but allow a small buffer.

These are starting points. Tighten if the matched data shows lower variance; widen only if a clear cluster of matches sits outside the default range.
