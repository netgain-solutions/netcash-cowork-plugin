# SuiteQL fallback patterns

Read-only SuiteQL templates for `ad-hoc-reporting` questions that the canned NetCash tools cannot express. Use only after confirming no canned tool covers the case.

## Universal safety rules

These rules apply to every template below and to any SuiteQL you write outside the templates:

1. **Read-only only.** SuiteQL via `ns_runCustomSuiteQL` is read-only by NetSuite design — no `INSERT`, `UPDATE`, `DELETE`, or DDL. Never construct write SQL. If the user asks for a mutation, route them to a NetCash write tool (`submitNetCash*`, `createNetCash*`, etc.) or to the appropriate skill (`matching`, `automation-rules`).
2. **Parameterize values.** Never inline user-supplied values directly into the SQL string. Use `?` placeholders + a parameter array, or sanitize the value to a known-safe shape (e.g., enforce ISO date, numeric amount, integer id).
3. **Escape quotes in literals.** If you must inline a string (e.g., a hardcoded category name), escape single quotes by doubling them (`'O''Brien'`). Prefer parameters.
4. **Bound the result.** Always include a `WHERE` clause that bounds by date or account — unbounded queries against `transaction` are slow and may time out. Add `ROWNUM <= N` (or use `FETCH FIRST N ROWS ONLY` if supported) when the user wants a top-N view.
5. **Cite columns by name, not position.** No `SELECT *` — be explicit so the result shape is predictable for downstream formatting.
6. **One sentence on why.** Whenever you fall back to SuiteQL, state in chat what canned tool you tried and why it didn't fit. This is the deliberate-exception rule from the SKILL.md Hard Constraints section.

---

## Template 1 — Cross-table join: bank activity + entity master + custom segment

**When to use:** User asks for bank transactions enriched with custom-segment fields or an entity master attribute (e.g., vendor category, customer tier) that isn't on the bank-activity record.

```sql
SELECT
  ba.id              AS bank_activity_id,
  ba.trandate        AS transaction_date,
  ba.amount          AS amount,
  ba.description     AS description,
  v.entityid         AS vendor_id,
  v.companyname      AS vendor_name,
  v.category         AS vendor_category,
  cs.name            AS custom_segment_value
FROM
  customrecord_ngob_banktran ba
  LEFT JOIN vendor v
    ON ba.custrecord_ngob_banktran_entity = v.id
  LEFT JOIN customlist_xyz_segment cs
    ON ba.custrecord_ngob_banktran_segment = cs.id
WHERE
  ba.trandate BETWEEN ? AND ?
  AND ba.custrecord_ngob_banktran_account = ?
ORDER BY
  ba.trandate DESC
```

**Parameters:** `dateFrom`, `dateTo`, `accountId`. Confirm exact bank-activity table name and field IDs with `getNetCashFieldsForTable` before running — IDs above are illustrative.

---

## Template 2 — Time-series: same metric over rolling 12 months

**When to use:** User asks "trend of [metric] over the last 12 months" and wants a row per month. `getNetCashBankActivityByGrouping` can group by month, but if you need the monthly value of a derived metric (net inflow minus outflow, fee differential, etc.) the join + arithmetic is cleaner in SuiteQL.

```sql
SELECT
  TO_CHAR(trandate, 'YYYY-MM') AS month,
  SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END) AS total_inflow,
  SUM(CASE WHEN amount < 0 THEN amount ELSE 0 END) AS total_outflow,
  SUM(amount)                                       AS net_change,
  COUNT(*)                                          AS transaction_count
FROM
  customrecord_ngob_banktran
WHERE
  trandate >= ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -12)
  AND trandate <  TRUNC(SYSDATE, 'MM')
  AND custrecord_ngob_banktran_account = ?
GROUP BY
  TO_CHAR(trandate, 'YYYY-MM')
ORDER BY
  month
```

**Parameters:** `accountId`. The date math (`ADD_MONTHS`, `TRUNC`) is server-side so the rolling window stays current. Bound by account always.

---

## Template 3 — Cumulative balance over time

**When to use:** User asks for running balance, end-of-day balance trend, or "what was our balance on [date range]". Canned tools return point-in-time snapshots; this gives the curve.

```sql
SELECT
  trandate                                 AS transaction_date,
  amount                                   AS daily_change,
  SUM(amount) OVER (
    ORDER BY trandate, id
    ROWS UNBOUNDED PRECEDING
  )                                        AS running_balance
FROM
  customrecord_ngob_banktran
WHERE
  trandate BETWEEN ? AND ?
  AND custrecord_ngob_banktran_account = ?
ORDER BY
  trandate, id
```

**Parameters:** `dateFrom`, `dateTo`, `accountId`. `ROWS UNBOUNDED PRECEDING` is the windowing clause that makes the cumulative sum work; if the SuiteQL dialect rejects it, fall back to a self-join. Always include the account filter — running balance across all accounts is meaningless.

---

## Template 4 — Saved searches (run directly; SuiteQL reconstruction is the fallback)

**When to use:** User says "run my saved search called X" or references a customer-built saved search by name.

**Primary path — run it directly** (saved-search tools verified working 2026-06-02): use the `ad-hoc-reporting` SKILL's "Saved searches" workflow — `ns_listSavedSearches` (find by name) → confirm the match → `ns_runSavedSearch` (paged via `range_start` / `range_end`). This returns the exact output the saved search produces; prefer it over reconstructing the search in SQL.

**Fallback — reconstruct in SuiteQL** only when the named search can't be found or won't run (e.g., the user describes a search they haven't actually saved, or `ns_runSavedSearch` errors on an unusual standalone search type):

1. Ask the user to describe the saved search's filters (or to share a screenshot of the criteria tab). Don't guess what the saved search does.
2. Translate the filters into a SuiteQL query against the appropriate base tables (`transaction`, `transactionLine`, `customrecord_ngob_banktran`, etc.).
3. Run via `ns_runCustomSuiteQL`.
4. If the saved search produces output the user cares about (specific columns, groupings, sort), mirror that shape in the SuiteQL result.

**Parameters:** Up to N transaction ids (validate they're integers before binding).

---

## Template 5 — Reverse-lookup: bank activity matched to a specific GL transaction

**When to use:** User has a GL journal entry id, transaction id, or invoice id and wants to know which bank transactions are matched to it. Canned tools filter from the bank side; this filters from the GL side.

```sql
SELECT
  ba.id,
  ba.trandate,
  ba.amount,
  ba.description,
  ba.custrecord_ngob_banktran_account
FROM
  customrecord_ngob_banktran        ba
  JOIN customrecord_ngob_bankmatch  bm
    ON bm.custrecord_ngob_bankmatch_banktran = ba.id
WHERE
  bm.custrecord_ngob_bankmatch_gl = ?
ORDER BY
  ba.trandate
```

**Parameters:** `glTransactionId`. The match-record table name and join field IDs are illustrative — confirm with `getNetCashTablesAndFields` before running.

---

## Template 6 — Vendor concentration: % of total spend per vendor over a window

**When to use:** User asks "what % of our spend goes to the top 5 vendors" or "vendor concentration risk over the last 12 months". `getNetCashBankActivityByGrouping` returns totals; this returns totals + percentage of total in one query.

```sql
WITH spend AS (
  SELECT
    custrecord_ngob_banktran_entity AS vendor_id,
    SUM(ABS(amount))                AS total_spend
  FROM
    customrecord_ngob_banktran
  WHERE
    trandate BETWEEN ? AND ?
    AND amount < 0   -- outflows only
  GROUP BY
    custrecord_ngob_banktran_entity
)
SELECT
  v.companyname              AS vendor_name,
  s.total_spend              AS total_spend,
  ROUND(
    100.0 * s.total_spend / SUM(s.total_spend) OVER (),
    2
  )                          AS pct_of_total
FROM
  spend s
  LEFT JOIN vendor v ON s.vendor_id = v.id
ORDER BY
  s.total_spend DESC
FETCH FIRST 10 ROWS ONLY
```

**Parameters:** `dateFrom`, `dateTo`. The window function (`SUM(...) OVER ()`) computes the grand total once for the percentage calculation.

---

## Template 7 — Multi-currency: convert all activity to functional currency

**When to use:** User asks for cross-currency totals on a multi-book or OneWorld instance. Canned tools return native-currency amounts; this converts to functional currency at transaction-date rate.

```sql
SELECT
  ba.id,
  ba.trandate,
  ba.amount                              AS native_amount,
  ba.custrecord_ngob_banktran_currency   AS native_currency,
  fx.exchangerate                        AS rate,
  ROUND(ba.amount * fx.exchangerate, 2)  AS functional_amount
FROM
  customrecord_ngob_banktran ba
  LEFT JOIN consolidatedexchangerate fx
    ON  fx.fromcurrency  = ba.custrecord_ngob_banktran_currency
    AND fx.tocurrency    = ?  -- functional currency id
    AND fx.postingperiod = (
      SELECT id FROM accountingperiod
      WHERE startdate <= ba.trandate AND enddate >= ba.trandate
        AND ROWNUM = 1
    )
WHERE
  ba.trandate BETWEEN ? AND ?
  AND ba.custrecord_ngob_banktran_account = ?
ORDER BY
  ba.trandate
```

**Parameters:** `functionalCurrencyId`, `dateFrom`, `dateTo`, `accountId`. The exchange-rate join is dialect-sensitive — adjust if the customer is on a non-OneWorld instance (no `consolidatedexchangerate`) or uses a different rate type.

---

## Template 8 — Anomaly hunt: transactions exceeding 3x trailing-90-day stddev for the same merchant

**When to use:** User asks "what looks unusual" or "any outlier transactions". This is a `pattern-insights` workflow — defer to that skill if the user wants ranked insights. But for a one-off "is this $50K from Acme weird?" question, run this pattern.

```sql
WITH merchant_stats AS (
  SELECT
    custrecord_ngob_banktran_entity         AS entity_id,
    AVG(ABS(amount))                        AS mean_amt,
    STDDEV(ABS(amount))                     AS stddev_amt
  FROM
    customrecord_ngob_banktran
  WHERE
    trandate BETWEEN ? AND ?  -- 90-day baseline window
  GROUP BY
    custrecord_ngob_banktran_entity
  HAVING
    COUNT(*) >= 3   -- need at least 3 obs for stddev to be meaningful
)
SELECT
  ba.id,
  ba.trandate,
  ba.amount,
  v.companyname AS vendor_name,
  ms.mean_amt   AS baseline_mean,
  ms.stddev_amt AS baseline_stddev
FROM
  customrecord_ngob_banktran ba
  JOIN merchant_stats ms
    ON ms.entity_id = ba.custrecord_ngob_banktran_entity
  LEFT JOIN vendor v
    ON v.id = ba.custrecord_ngob_banktran_entity
WHERE
  ba.trandate BETWEEN ? AND ?  -- the lookback window for which to flag outliers
  AND ABS(ba.amount) > ms.mean_amt + 3 * ms.stddev_amt
ORDER BY
  ABS(ba.amount) DESC
```

**Parameters:** baseline `dateFrom`/`dateTo` (90 days), evaluation `dateFrom`/`dateTo`. Flag this is descriptive, not predictive — the threshold is the same one used in `../../pattern-insights/references/anomaly-thresholds.md`.

---

## When NOT to use SuiteQL

If your question fits any of these shapes, stay with the canned tool — even if SuiteQL feels faster:

- **Filtered list of bank transactions** → `getNetCashBankActivity`. Even with five filters, the canned tool handles it.
- **Aggregate by one dimension** (vendor, month, account) → `getNetCashBankActivityByGrouping`.
- **Cash position / account & GL balances** → `getNetCashAccountBalances`. **Cash-flow roll-up (spend/income trend)** → `getNetCashBankActivitySummary`.
- **Open AR / open AP** → `getNetCashCashApplicationTransactions`.
- **Recon status** → `getNetCashReconciliations`.

The cost of using SuiteQL when a canned tool would have worked: payload bloat, drift risk if NetCash schema changes, and you skip the canned tool's built-in security/RBAC behavior. Stick to canned unless you have a real reason not to.
