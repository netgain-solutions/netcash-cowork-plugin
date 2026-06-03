# Diagnostic Queries

SuiteQL templates for advanced NetCash diagnostics — used only when the canned NetCash MCP tools (`getNetCashReportMatches`, `getNetCashBankActivity`, `getNetCashGLActivity`, `getNetCashBankActivityByGrouping`) cannot express the question. All queries are read-only. Run via `ns_runCustomSuiteQL` (last-resort fallback per the plugin-level tool-routing principle).

When using a NetCash canned tool would work, use it instead. These queries exist for the cases where the canned surface doesn't reach.

---

## Orphaned matches

A match record (`customrecord_ngob_bankmatch`) with fewer than two child transaction lines (`customrecord_ngob_bankmatchtran`) is structurally orphaned — a match should always pair at least one bank line with at least one GL line.

```sql
SELECT
    bm.id,
    bm.name,
    COUNT(bmt.id) AS line_count
FROM customrecord_ngob_bankmatch bm
LEFT JOIN customrecord_ngob_bankmatchtran bmt
    ON bmt.custrecord_ngob_bankmatchtran_match = bm.id
GROUP BY bm.id, bm.name
HAVING COUNT(bmt.id) < 2
ORDER BY bm.id DESC
```

**Use when:** the user reports a "ghost" match (visible in the recon UI but with no apparent transactions on either side), or after a bulk-unmatch operation that may have left orphans.

**Remediation if orphans found:** depends on whether the orphan can be safely cleaned up. Don't recommend deletion without checking dependent records first (see SKILL.md hard constraints). Inactivate is usually the safer recommendation.

---

## Amount mismatches on existing matches

A match where the sum of bank-side amounts does not equal the sum of GL-side amounts (within a penny). Either the match was created with mismatched totals (a write-time defect) or one side was modified post-match.

```sql
SELECT
    bm.id,
    bm.name,
    SUM(CASE
        WHEN bmt.custrecord_ngob_bankmatchtran_isbank = 'T'
        THEN bmt.custrecord_ngob_bankmatchtran_amount ELSE 0
    END) AS bank_total,
    SUM(CASE
        WHEN bmt.custrecord_ngob_bankmatchtran_isbank = 'F'
        THEN bmt.custrecord_ngob_bankmatchtran_amount ELSE 0
    END) AS gl_total
FROM customrecord_ngob_bankmatch bm
JOIN customrecord_ngob_bankmatchtran bmt
    ON bmt.custrecord_ngob_bankmatchtran_match = bm.id
GROUP BY bm.id, bm.name
HAVING ABS(
    SUM(CASE
        WHEN bmt.custrecord_ngob_bankmatchtran_isbank = 'T'
        THEN bmt.custrecord_ngob_bankmatchtran_amount ELSE 0
    END)
    - SUM(CASE
        WHEN bmt.custrecord_ngob_bankmatchtran_isbank = 'F'
        THEN bmt.custrecord_ngob_bankmatchtran_amount ELSE 0
    END)
) > 0.01
ORDER BY ABS(bank_total - gl_total) DESC
```

**Use when:** the user's recon balance doesn't tie out and you suspect a previously-matched item has drifted (someone modified an amount post-match, or a match was written with an unaccounted variance).

**Tolerance note:** the `> 0.01` threshold catches floating-point rounding. If the customer has a higher rounding tolerance (e.g., for FX), increase the threshold accordingly — see `multi-currency.md` for FX rounding logic.

---

## PSP variance detection (candidate Stripe / PayPal / Square deposits)

Bank deposits whose description matches a PSP signature and which remain unmatched. Used to find candidates for PSP-fee variance handling before the user asks about them.

```sql
SELECT
    bt.id,
    bt.custrecord_ngob_banktran_date AS posting_date,
    bt.custrecord_ngob_banktran_amount AS amount,
    bt.custrecord_ngob_banktran_description AS description,
    bt.custrecord_ngob_banktran_account AS account
FROM customrecord_ngob_banktran bt
WHERE bt.custrecord_ngob_banktran_matched = 'F'
  AND bt.isinactive = 'F'
  AND bt.custrecord_ngob_banktran_amount > 0
  AND (
      UPPER(bt.custrecord_ngob_banktran_description) LIKE '%STRIPE%'
      OR UPPER(bt.custrecord_ngob_banktran_description) LIKE '%PAYPAL%'
      OR UPPER(bt.custrecord_ngob_banktran_description) LIKE '%SQUARE%'
      OR UPPER(bt.custrecord_ngob_banktran_description) LIKE '%SQ *%'
  )
ORDER BY bt.custrecord_ngob_banktran_date DESC
```

**Use when:** the user reports an "amount mismatch" on a deposit and the PSP signature isn't obvious from a single bank-row inspection. Also useful before a bulk PSP-rule build via `automation-rules`.

**Follow-up:** for each candidate, calculate the expected net per the PSP fee structure in `known-failure-modes.md` § PSP variance, then check whether an open invoice in the same currency exists at the calculated gross. Match with variance if confirmed.

---

## Duplicate bank transactions (same date / amount / merchant)

Used to confirm a suspected duplicate before recommending inactivation. NetCash imports can occasionally double-import on retry; this query catches the obvious cases.

```sql
SELECT
    bt.custrecord_ngob_banktran_date AS posting_date,
    bt.custrecord_ngob_banktran_amount AS amount,
    bt.custrecord_ngob_banktran_merchant AS merchant,
    COUNT(*) AS occurrences,
    LISTAGG(bt.id, ',') WITHIN GROUP (ORDER BY bt.id) AS ids
FROM customrecord_ngob_banktran bt
WHERE bt.isinactive = 'F'
  AND bt.custrecord_ngob_banktran_date >= SYSDATE - 60
GROUP BY
    bt.custrecord_ngob_banktran_date,
    bt.custrecord_ngob_banktran_amount,
    bt.custrecord_ngob_banktran_merchant
HAVING COUNT(*) > 1
ORDER BY bt.custrecord_ngob_banktran_date DESC
```

**Use when:** the user reports a duplicate match. Run this first to confirm the duplicate is on the bank side (not a duplicate match on the same single transaction — those need the orphaned-matches query).

**Note:** prefer `getNetCashBankActivityByGrouping` for the same question if the canned tool covers the user's date range and grouping.

---

## Notes on safety

- All queries above are SELECT-only. No `UPDATE`, `INSERT`, `DELETE`, or `MERGE`. Confirm before pasting any query into a NetSuite SuiteQL workbench.
- Always run in a development environment first when possible. Production SuiteQL has no transaction guard — query results can drive real remediation actions.
- When passing user-supplied values into a query (account name, date, amount), parameterize them rather than string-concatenating. NetSuite's SuiteQL endpoint supports parameterized queries via `ns_runCustomSuiteQL`.
- Field IDs are accurate as of the schema introspected at plugin authoring time. If a query returns an unexpected error, run `getNetCashFieldsForTable` against the relevant table to confirm field IDs haven't drifted.
