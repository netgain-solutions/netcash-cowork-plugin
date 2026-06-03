# Rule schemas — full JSON for all four NetCash automation rule types

Reference for the `automation-rules` skill. Every `ruleJson` payload conforms to one of these four schemas. Before calling any `createNetCash*Rule`, validate via `validateNetCashAutomationRuleFields`.

Field IDs in examples are illustrative — always pull the live list from `getNetCashAutomationRuleFieldOptions` and substitute. Never paste these IDs into a `ruleJson` without verification.

## Common defaults across all types

- `status: 2` (Testing) — required, never override.
- `isRequireReview: true` — required, never override.
- `group: []` — assigned to one or more rule groups (from `getNetCashAutomationRuleGroups`). Default to the group(s) covering the account in scope; use all if uncertain.
- `filters: []` — array of filter objects (see filter structure below).
- `source` on filters: `1` = bank, `2` = GL.

### Filter structure (all types)

```json
{ "source": 1, "field": "<bank or GL field ID>", "operator": <1-7>, "value": "STRING_OR_NUMBER" }
```

Make filters specific enough to avoid false positives. CONTAINS (operator 2) is the default for text — bank data is noisy. See `operator-reference.md` for operator IDs.

---

## Type 1 — GL Match

Matches bank transactions to existing GL transactions (bill payments, deposits, prior JEs). Filters narrow the candidate GL pool; `matchCriteria` pair a specific bank transaction to a specific GL transaction.

GL Match rules should almost always include both filters and match criteria. A filters-only GL Match rule is technically valid but rarely useful — flag it as unusual if proposing one.

```json
{
  "name": "string",
  "type": 1,
  "status": 2,
  "group": [],
  "amountVariance": 0,
  "isAmountPercentage": false,
  "dateTolerance": 3,
  "isRequireReview": true,
  "isRequireAll": true,
  "filters": [],
  "matchCriteria": [],
  "groupingCriteria": []
}
```

### `matchCriteria` structure

```json
{ "bankField": "<bank field ID>", "operator": 2, "glField": "<GL field reference>", "isMatchNulls": false }
```

- `isRequireAll`: `true` = ALL match criteria must hold (AND); `false` = ANY match criterion holds (OR).
- `isMatchNulls`: only set `true` when null-to-null pairings are explicitly observed.

### Example `ruleJson` payload

```json
{
  "ruleJson": "{\"name\":\"ABC Vendor - Check Match\",\"type\":1,\"status\":2,\"group\":[],\"amountVariance\":0,\"isAmountPercentage\":false,\"dateTolerance\":5,\"isRequireReview\":true,\"isRequireAll\":true,\"filters\":[{\"source\":1,\"field\":\"<bank merchant field>\",\"operator\":2,\"value\":\"ABC VENDOR\"}],\"matchCriteria\":[{\"bankField\":\"<bank check number field>\",\"operator\":2,\"glField\":\"transaction.otherrefnum\",\"isMatchNulls\":false}],\"groupingCriteria\":[]}"
}
```

### Type 1 boundaries

- No `isMatchNulls: true` unless null-to-null matches are observed.
- No amount-only matching without filters.
- No `isRequireAll: false` without explicit OR-logic justification.

---

## Type 2 — Create Transaction

Creates new GL entries for bank activity that has no existing GL counterpart (fees, interest, subscriptions, dividends, ATM withdrawals).

Use Type 2 when match history shows the user manually created JEs to reconcile bank transactions. Indicators: JE created on or after the bank transaction date, generic memos like "bank fee" / "wire fee" / "interest" / "dividend", no matching vendor bill or deposit.

```json
{
  "name": "string",
  "type": 2,
  "status": 2,
  "group": [],
  "transactionType": 1,
  "filters": [],
  "transactionTemplateLines": []
}
```

- `transactionType`: `1` = Journal Entry (default), `2` = Check/Deposit.
- `filters`: required for Type 2. Without filters the rule fires on every transaction.

### `transactionTemplateLines` shapes

The MCP `createNetCashCreateTransactionRule` tool exposes only these fields per template line: `source` (1 = Bank Account line, 2 = Offset Account line), `account`, `memo`, `entity`, `taxCode`. **Department, Class, Location, and Business Unit (and any other custom segments) are not exposed by the MCP tool** — they must be set via the NetCash UI on the rule's Transaction Template tab after the rule is created. Verified against the live MCP `createNetCashCreateTransactionRule` schema, which enumerates exactly those five fields.

- Minimal (memo + GL account set later in NetCash): `[{ "source": 1 }]`
- With memo: `[{ "source": 1, "memo": "Monthly bank fee" }]`
- With account: `[{ "source": 1, "account": 123, "memo": "Fee" }]`
- With entity (e.g., vendor on the offset line for AP-side Create Transaction): `[{ "source": 1, "account": 123, "memo": "Fee", "entity": 456 }]`
- With tax code: `[{ "source": 1, "account": 123, "memo": "Fee", "taxCode": 7 }]`

When the user supplies a screenshot or template that includes Department / Class / Location / Business Unit values, resolve as much as you can (account, entity, tax code) via the MCP, then surface the segment values you couldn't pass programmatically — and tell the user explicitly that they'll need to populate Department / Class / Location / Business Unit on the rule's Transaction Template tab in NetCash before activating. Do not silently drop them; do not silently leave them blank.

### Example `ruleJson` payload

```json
{
  "ruleJson": "{\"name\":\"Money Market Fund Dividend Income\",\"type\":2,\"status\":2,\"group\":[],\"transactionType\":1,\"filters\":[{\"source\":1,\"field\":\"<bank description field>\",\"operator\":2,\"value\":\"MMF\"},{\"source\":1,\"field\":\"<bank description field>\",\"operator\":2,\"value\":\"Dividend\"}],\"transactionTemplateLines\":[{\"source\":1,\"memo\":\"Money Market Fund Dividend Income\"}]}"
}
```

### Type 2 boundaries

- Don't suggest Type 2 for transactions that should be GL Match or Cash Application.
- Don't guess GL account IDs — use the minimal template if the account isn't deterministic from the matched data, and tell the user to set it in NetCash before activating.

---

## Type 3 — Create Transfer

Links bank-to-bank transfers across accounts in the same instance.

```json
{
  "name": "string",
  "type": 3,
  "status": 2,
  "group": [],
  "isCopyIntercompanyValues": false,
  "filters": []
}
```

### Filters with flow direction

```json
{ "source": 1, "field": "<bank description field>", "operator": 2, "value": "TRANSFER", "flow": 3 }
```

- `flow`: `1` = inflow only, `2` = outflow only, `3` = both (default, recommended). Both is correct for most transfers because the rule needs to recognize each side.

### Transfer-specific fields

- `isCopyIntercompanyValues` — copies department / class / location to the receiving side for intercompany transfers.
- `custrecord_ngob_bank_rule_ic_receve_acct` — IC receivables account override.
- `custrecord_ngob_bank_rule_ic_payabl_acct` — IC payables account override.

Common keywords seen in transfer descriptions: "transfer", "trnf", "trfs", "xfer", "txfr".

### Example `ruleJson` payload

```json
{
  "ruleJson": "{\"name\":\"Operating to Reserve Transfer\",\"type\":3,\"status\":2,\"group\":[],\"isCopyIntercompanyValues\":false,\"filters\":[{\"source\":1,\"field\":\"<bank description field>\",\"operator\":2,\"value\":\"TRANSFER\",\"flow\":3}]}"
}
```

### Type 3 boundaries

- Don't suggest Type 3 for vendor payments or customer receipts.
- Don't use overly broad filters — "TRANSFER" alone catches vendor wire transfers in many bank feeds.
- Don't enable intercompany unless the matched pattern shows it.
- Verify both sides exist (an outflow on one account and a corresponding inflow on the other). A one-sided "transfer" pattern is usually a Type 1 or Type 2.

---

## Type 4 — Cash Application

Creates payment transactions to apply bank deposits to open invoices (AR) or bank withdrawals to open bills (AP).

```json
{
  "name": "string",
  "type": 4,
  "status": 2,
  "group": [],
  "amountVariance": 0,
  "isAmountPercentage": false,
  "dateField": true,
  "dateTolerance": 30,
  "isRequireReview": true,
  "isRequireAll": true,
  "custrecord_ngob_bank_rule_is_apply_disct": false,
  "custrecord_ngob_bank_rule_is_bk_rounding": false,
  "filters": [],
  "matchCriteria": [],
  "groupingCriteria": []
}
```

### Special fields

- `dateField` — Type 4 only. Boolean controlling which GL date anchors the tolerance window. `true` anchors on the invoice/bill **due date** (falls back to tran date if no due date exists); `false`/omitted anchors on the GL transaction date. Default to `true` for both AR and AP rules — bank deposits arrive near invoice due date, and bank withdrawals against bills cluster near bill due date, not issue date. Default to `false` for PSP rules (Stripe/PayPal/Square) — payouts are anchored on payout date, not customer due dates. See `known-patterns.md` Section 2 for the default rationale.
- `custrecord_ngob_bank_rule_is_apply_disct` — apply early-payment discounts.
- `custrecord_ngob_bank_rule_is_bk_rounding` — generate a rounding journal for overpayments.

### Direction filters

- Customer payments (deposits): `{ "source": 1, "field": "<bank credit field>", "operator": 6, "value": "0" }` (greater than 0).
- Vendor payments (withdrawals): `{ "source": 1, "field": "<bank debit field>", "operator": 6, "value": "0" }` (greater than 0).

### Payment-processor handling

For Stripe, PayPal, Square: set `isAmountPercentage: true` with `amountVariance: 4-5` to absorb 2.9% (Stripe/PayPal) or 2.6% (Square) fees. See `known-patterns.md` for worked PSP examples.

### Type 4 grouping

Use `groupingCriteria` only when one bank transaction covers multiple open items for the same customer or vendor.

- One deposit → multiple invoices for the same customer:
  ```json
  { "groupingCriteria": [{ "source": 2, "field": "NVL(transactionLine.entity,transaction.entity)" }] }
  ```
- Stripe-style batch deposits (multiple customers in one deposit): use `transaction.memo` instead, or omit `groupingCriteria`.

Same cardinality rule as Type 1: only group on a field that's actually shared across all GL records in the matchgroup. See `grouping-criteria.md`.

### Example `ruleJson` payload

```json
{
  "ruleJson": "{\"name\":\"Stripe Cash Application\",\"type\":4,\"status\":2,\"group\":[],\"amountVariance\":4,\"isAmountPercentage\":true,\"dateTolerance\":5,\"isRequireReview\":true,\"isRequireAll\":true,\"custrecord_ngob_bank_rule_is_apply_disct\":false,\"custrecord_ngob_bank_rule_is_bk_rounding\":false,\"filters\":[{\"source\":1,\"field\":\"<bank merchant field>\",\"operator\":2,\"value\":\"STRIPE\"},{\"source\":1,\"field\":\"<bank credit field>\",\"operator\":6,\"value\":\"0\"}],\"matchCriteria\":[{\"bankField\":\"<bank description field>\",\"operator\":2,\"glField\":\"transaction.memo\",\"isMatchNulls\":false}],\"groupingCriteria\":[{\"source\":2,\"field\":\"transaction.memo\"}]}"
}
```

### Type 4 boundaries

- Don't enable discount or rounding without matched evidence.
- Don't use Type 4 for transactions that already have payments applied (those are Type 1 GL Match).
- Don't use excessive amount variance — keep PSP fees inside `isAmountPercentage` rather than wide flat ranges.

---

## Fallback SuiteQL templates

When the canned tools are unavailable, these read-only queries can substitute. They live here so the SKILL.md body stays focused on the workflow.

### List active rules with type + status

```sql
SELECT
  id,
  name,
  custrecord_ngob_bank_rule_type AS type,
  custrecord_ngob_bank_rule_status AS status
FROM customrecord_ngob_bank_rule
WHERE isinactive = 'F'
```

Type values: `1` = GL Match, `2` = Create Transaction, `3` = Create Transfer, `4` = Cash Application.

### List filter detail per rule

```sql
SELECT
  r.id,
  r.name,
  f.custrecord_ngob_rulefilter_field AS field,
  f.custrecord_ngob_rulefilter_operator AS operator,
  f.custrecord_ngob_rulefilter_value AS value
FROM customrecord_ngob_bank_rule r
LEFT JOIN customrecord_ngob_rule_filter f
  ON f.custrecord_ngob_rulefilter_rule = r.id
WHERE r.isinactive = 'F'
```

### Resolve account internal ID to display name

```sql
SELECT id, accountsearchdisplayname, acctnumber
FROM account
WHERE id = <account_id>
```

These are fallbacks only — `getNetCashAutomationRules` is the preferred path when available.
