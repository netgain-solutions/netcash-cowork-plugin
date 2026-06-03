# Grouping criteria — when and how to use `groupingCriteria` on NetCash automation rules

Reference for the `automation-rules` skill. Use this when proposing GL Match (Type 1) or Cash Application (Type 4) rules that involve many-to-one or one-to-many matchgroups. Type 2 and Type 3 do not use `groupingCriteria`.

## When to use grouping

Inspect the matchgroup data before suggesting `groupingCriteria`. The signal is straightforward:

- **One bank transaction → many GL records.** The user manually applied one bank deposit to multiple invoices, or one bank payroll deposit to multiple journal lines. Group on the **GL side** (`source: 2`).
- **Many bank transactions → one GL record.** Multiple daily fees roll up to one monthly summary GL entry. Group on the **bank side** (`source: 1`).
- **Many bank transactions → many GL records (M:M).** Both sides collapse into groups before pairing — e.g., a recurring vendor pattern where the number of bank legs varies per cycle AND the GL side has multiple journal lines per cycle. Group on **both sides** (`source: 1` for bank-side keys, `source: 2` for GL-side keys). The engine builds each side's group independently and then pairs group totals against each other. M:M is fully supported by the matching engine — see "Many-to-many (M:M)" example below.
- **One-to-one pairings.** Omit `groupingCriteria` entirely. Adding a grouping field to a 1:1 rule produces single-record groups, which has no effect — and adds visual noise that obscures rule intent in NetCash.

## The cardinality rule (the rule that matters most)

The grouping field must be **shared across all records in the group**. If every record has a different value in the candidate field, the rule produces single-record groups and the grouping has no effect.

Before choosing a grouping field, do this verification:

1. Pull sample GL (or bank) records from a representative matchgroup in the matched data.
2. For each candidate field, check whether multiple records in the same matchgroup share the same value.
3. If no field is shared across all records in the group, do not use `groupingCriteria`.

Skipping this verification is the most common reason a rule "looks right" in the proposal but never groups in production.

## Good grouping examples

### One bank → many GL, all share entity

Single $48,300 Stripe deposit → 12 customer invoices, all customer = "Acme Corp". Grouping field shared across all 12 GL lines:

```json
{ "groupingCriteria": [{ "source": 2, "field": "NVL(transactionLine.entity, transaction.entity)" }] }
```

Works because all GL lines share the entity.

### One bank → many GL, all share memo

Single ADP payroll deposit → 8 journal lines, all share `transaction.memo` = "ADP payroll":

```json
{ "groupingCriteria": [{ "source": 2, "field": "transaction.memo" }] }
```

Works because the memo is the shared field across the journal lines.

### Many bank → one GL, all share date

22 daily card-network fees → one monthly summary JE. Each daily fee shares the same `custrecord_ngob_banktran_date` only when grouped by month — but if the day of month is the unifier, the bank date works. More commonly, a derived month-key or the merchant string is the unifier.

For daily-fee → monthly-summary patterns, the bank merchant or category is usually the safer grouping field:

```json
{ "groupingCriteria": [{ "source": 1, "field": "<bank merchant field>" }] }
```

### Many-to-many (M:M) — both sides grouped

A recurring payroll/funding pattern where the number of bank-side legs varies per cycle (e.g., 3 wires one week, 11 the next) and the GL side has multiple journal-entry lines per cycle. Each side has its own shared key — bank legs share a merchant string, GL lines share an entity ID. Group both sides:

```json
{
  "groupingCriteria": [
    {"source": 1, "field": "custrecord_ngob_banktran_merchant"},
    {"source": 2, "field": "NVL(transactionLine.entity, transaction.entity)"}
  ]
}
```

The engine processes each side's grouping independently — bank-side SQL collapses the N wires sharing the merchant string into one row whose `amount` is the sum, GL-side SQL collapses the M JE lines sharing the entity into one row whose `amount` is the sum — then the matching engine pairs the two grouped rows on amount and date proximity within `dateTolerance`. The cardinality rule applies on **both sides**: the bank-side grouping field must be shared across all bank legs in a cycle, and the GL-side grouping field must be shared across all GL lines in the same cycle. If either cardinality fails, the engine produces single-record groups on that side and the rule degrades to 1:M or M:1 behavior on the other side.

M:M grouping is fully supported by the NetCash matching engine — the bank-side and GL-side grouped sets are built independently and then fed to the matching pipeline simultaneously. There is no "one side must be ungrouped" constraint. Confirm with Netgain Support before relying on M:M in production if you are unsure how a specific rule will behave.

### One-to-one (no grouping)

Vendor wire to a single bill: one bank transaction, one GL bill payment. Omit `groupingCriteria` entirely.

## Bad grouping examples

### Each GL line has a different entity

12 GL lines from a Stripe batch → 12 different customers. `entity` is unique per line:

```json
{ "groupingCriteria": [{ "source": 2, "field": "NVL(transactionLine.entity, transaction.entity)" }] }
```

This produces 12 single-record groups. Same outcome as no grouping. Either use a different shared field (`transaction.memo` if all 12 lines share the Stripe deposit memo) or omit `groupingCriteria`.

### Grouping by `transaction.tranId`

Every transaction has a unique `tranId`. Grouping on it always produces single-record groups:

```json
{ "groupingCriteria": [{ "source": 2, "field": "transaction.tranId" }] }
```

Don't do this. `tranId` is invalid as a grouping field.

### Inferring a shared field that isn't actually shared

Sample GL records all happen to have memo = "ACME Corp" in one matchgroup, but other matchgroups have memo = "ABC Corp" or empty. The field is shared *within* a matchgroup but not deterministic *across* the rule's expected input. This is fine — the cardinality rule applies per-matchgroup, not across the whole rule. Verify the within-group sharing on multiple sample matchgroups, not just one.

## Valid grouping fields by source

These are the most common valid fields. Always cross-check against `getNetCashAutomationRuleFieldOptions` for the live list.

### GL side (source: 2)

Common:
- `transaction.trandate` — group by GL post date.
- `NVL(transactionLine.entity, transaction.entity)` — group by entity (customer / vendor).

Less common but valid:
- `transaction.memo` — group by header memo.
- `transactionLine.memo` — group by line memo.
- `transactionAccountingLine.account` — group by GL account.

### GL side — invalid

- `transaction.tranId` — unique per transaction. Always produces single-record groups.

### Bank side (source: 1)

- `custrecord_ngob_banktran_date` — group by bank date.
- `custrecord_ngob_banktran_merchant` — group by merchant string.
- `custrecord_ngob_banktran_category_primar` — group by primary bank category.
- `custrecord_ngob_banktran_description` — group by description (less reliable; bank descriptions are noisy and may not share exactly).

## Worked example — Stripe batch deposit

User pattern: one Stripe deposit on the bank side, applied to 8 customer invoices on the GL side. All 8 invoices share `transaction.memo` containing the Stripe payout ID; entity differs per invoice.

Wrong: group on entity. Each of the 8 invoices has a different customer → 8 single-record groups, no actual grouping.

Right: group on `transaction.memo`. All 8 invoices share the Stripe payout ID in their memo → 1 group of 8.

```json
{
  "groupingCriteria": [{ "source": 2, "field": "transaction.memo" }]
}
```

## Worked example — payroll batch

User pattern: one ADP payroll bank withdrawal, applied to 1 GL journal entry with 12 lines (one per employee or per cost center). All 12 lines share `transaction.memo` = "ADP payroll" and `transactionAccountingLine.account` is varied.

Right: group on `transaction.memo`.

```json
{
  "groupingCriteria": [{ "source": 2, "field": "transaction.memo" }]
}
```

Wrong: group on `transactionAccountingLine.account`. The GL accounts vary per line.

## Worked example — monthly summary fee JE

User pattern: 22 daily card-processing fees on the bank side roll up to one monthly summary JE on the GL side. The daily bank fees all share `custrecord_ngob_banktran_merchant` = "VISA NETWORK FEE" or similar.

Right: group on bank merchant.

```json
{
  "groupingCriteria": [{ "source": 1, "field": "<bank merchant field>" }]
}
```

The cardinality is many bank → one GL, so source = 1 (bank side), and the merchant is shared across all 22 bank fees.

## Verification checklist before adding `groupingCriteria`

1. Identify the cardinality direction (one bank → many GL, or many bank → one GL).
2. Identify candidate fields shared across all records in the group.
3. Pull at least 2 sample matchgroups from the matched data and confirm the candidate field is shared in each.
4. If no shared field exists across all records of all sample matchgroups, omit `groupingCriteria`.
5. If the cardinality is 1:1, always omit `groupingCriteria`.
