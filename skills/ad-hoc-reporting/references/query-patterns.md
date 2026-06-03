# Query patterns — natural-language to filter parameters

This reference maps ~20 common controller questions to the canned NetCash tool + parameters that answer them. Use as a lookup when parsing a user query in the `ad-hoc-reporting` skill.

For every example: parameter values shown are illustrative shapes, not literal field names. Confirm exact field IDs with `getNetCashFieldsForTable` if the call fails or a field name doesn't match.

---

## `columnSet` modes for `getNetCashBankActivity`

`getNetCashBankActivity` exposes five `columnSet` modes. Pick the leanest one that answers the question — do not default to `all`. Smaller column sets reduce tool-call payload size (faster, less noisy in chat).

| Mode | Columns returned (approximate) | Use when |
|---|---|---|
| `minimal` | id, date, amount, description | Quick lookup; user just wants to confirm a transaction exists. |
| `default` | minimal + match status, account name, transaction type | The default for most filtered-list questions. Start here. |
| `detailed` | default + entity (vendor/customer), memo, transaction-type detail, check number | User is investigating — needs context to decide what's going on. |
| `financial` | default + currency, exchange rate, FX-source amount, FX gain/loss-relevant fields | Multi-currency question; user is comparing across currencies or asking about FX. |
| `all` | Every available column on the bank-activity record | User asked for "everything" or you genuinely don't know which columns matter yet — fall back when the others are insufficient. |

When in doubt, pick `default`. Escalate to `detailed` if the user asks "why" or "what's this?". Escalate to `financial` only when currency or FX is part of the question.

---

## 20 worked examples

### 1. "Show me last 30 days of unmatched ACH > $10K from Stripe"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `dateFrom`: today minus 30 days
- `dateTo`: today
- `transactionType`: ACH
- `amountFrom`: 10000
- `isOnlyUnmatched`: true
- `descriptionContains`: "stripe" (case-insensitive)
- `columnSet`: `detailed` (user wants to see why these aren't matched)

### 2. "Biggest expense vendor this month"
**Tool:** `getNetCashBankActivityByGrouping`
**Parameters:**
- `dateFrom`: first of current month
- `dateTo`: today
- `direction`: outflow / debit
- `groupBy`: entity (vendor)
- `orderBy`: total amount descending
- `limit`: 1 (or 10 if the user wants a top list)

### 3. "Compare deposits this month vs last month"
**Tool:** Two calls to `getNetCashBankActivity` (or `getNetCashBankActivityByGrouping` with month grouping if the user wants by-day breakdown).
**Parameters:**
- Call A: `dateFrom`/`dateTo` = current month, `direction`: inflow
- Call B: `dateFrom`/`dateTo` = prior month, `direction`: inflow
- `columnSet`: `default`
Format result as a side-by-side table with delta and delta-%.

### 4. "List all transactions on BOA in November"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `accountId`: the BOA account (look up via `getNetCashBankAccounts` if user only said "BOA")
- `dateFrom`: 2026-11-01 (or whichever November the user means — clarify if ambiguous)
- `dateTo`: 2026-11-30
- `columnSet`: `default`

### 5. "Top 10 vendors by spend YTD"
**Tool:** `getNetCashBankActivityByGrouping`
**Parameters:**
- `dateFrom`: first of current year
- `dateTo`: today
- `direction`: outflow
- `groupBy`: entity
- `orderBy`: total amount descending
- `limit`: 10

### 6. "What's the largest deposit on the operating account this week?"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `accountId`: operating account
- `dateFrom`: most recent Monday (or 7 days back)
- `dateTo`: today
- `direction`: inflow
- `orderBy`: amount descending
- `limit`: 1
- `columnSet`: `detailed`

### 7. "Show me wires over $50K in the last 90 days"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `dateFrom`: today minus 90 days
- `transactionType`: wire
- `amountFrom`: 50000
- `columnSet`: `detailed`

### 8. "How much did we pay AWS last quarter?"
**Tool:** `getNetCashBankActivityByGrouping`
**Parameters:**
- `dateFrom`/`dateTo`: prior calendar quarter
- `direction`: outflow
- `entityContains`: "amazon" or "aws"
- `groupBy`: entity (so you get one row per matched entity name)
Or: `getNetCashBankActivity` with the same filters and sum manually if the entity match isn't clean.

### 9. "Unmatched bank activity older than 30 days"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `isOnlyUnmatched`: true
- `dateTo`: today minus 30 days
- `orderBy`: date ascending (oldest first)
- `columnSet`: `detailed`

### 10. "Total ACH volume on operating account in November"
**Tool:** `getNetCashBankActivityByGrouping`
**Parameters:**
- `accountId`: operating account
- `transactionType`: ACH
- `dateFrom`/`dateTo`: November
- `groupBy`: account (returns one row with the total)
Single-answer format — return one sentence with the total.

### 11. "List all checks over $5K to Acme Corp this year"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `dateFrom`: first of current year
- `transactionType`: check
- `amountFrom`: 5000
- `entityContains`: "acme"
- `columnSet`: `detailed` (user likely wants check numbers and memos)

### 12. "Cash position across all accounts as of yesterday"
**Tool:** `getNetCashAccountBalances`
**Parameters:**
- `asOfDate`: yesterday (YYYY-MM-DD)
- `groupBy`: `bank` (collapse sibling accounts under one bank) or `account` (one row each)
Returns live Account Balance, GL Balance, and variance per bank **and** credit-card account, mirroring the NetCash dashboard balance widget — don't derive balances from `getNetCashBankActivitySummary` (that's a cash-flow tool). This is also a `recon-dashboards` question — suggest `/cash-position` if the user wants a recurring view.

### 13. "Open invoices over 60 days old"
**Tool:** `getNetCashCashApplicationTransactions`
**Parameters:**
- `transactionType`: invoice
- `status`: open
- `agedOver`: 60 (days)
- `orderBy`: age descending

### 14. "Open bills owed to Stripe"
**Tool:** `getNetCashCashApplicationTransactions`
**Parameters:**
- `transactionType`: bill
- `status`: open
- `entityContains`: "stripe"

### 15. "GL postings to account 1010 this month"
**Tool:** `getNetCashGLActivity`
**Parameters:**
- `accountId`: 1010 (look up the internal id if the user gave the account number/name)
- `dateFrom`/`dateTo`: current month
- `columnSet`: `default`

### 16. "Recon status across all accounts this period"
**Tool:** `getNetCashReconciliations`
**Parameters:**
- `period`: current period
- `status`: in-progress, not-started (exclude completed)
Suggest `/recon-status` for a recurring view.

### 17. "Stripe deposits with fee deductions this quarter"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `dateFrom`/`dateTo`: current quarter
- `direction`: inflow
- `descriptionContains`: "stripe"
- `columnSet`: `detailed`
Then in output, note that fee differential will be visible by comparing gross-payment hints in the description vs. the deposit amount. Suggest `pattern-insights` if user wants the systematic view.

### 18. "Transfers between accounts in the last 7 days"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `dateFrom`: today minus 7 days
- `transactionType`: transfer
- `columnSet`: `detailed` (user wants to see source + dest)

### 19. "Largest single outflow this year"
**Tool:** `getNetCashBankActivity`
**Parameters:**
- `dateFrom`: first of current year
- `direction`: outflow
- `orderBy`: amount descending (absolute value)
- `limit`: 1
- `columnSet`: `detailed`

### 20. "All transactions matched to GL Journal #12345"
**Tool:** Try `getNetCashBankActivity` with a `matchedGlReference` filter; if no such filter exists, fall back to `ns_runCustomSuiteQL` (this is a cross-table reverse-lookup that the canned tool may not express). See `suiteql-fallback-patterns.md` example "Reverse-lookup by GL transaction".

---

## Patterns to remember

- **Aggregation question?** → `getNetCashBankActivityByGrouping`. Don't fetch raw rows and sum in the model.
- **List question?** → `getNetCashBankActivity` with a tight `columnSet`.
- **GL side?** → `getNetCashGLActivity`.
- **Open AR/AP?** → `getNetCashCashApplicationTransactions`.
- **Cash position / balances across accounts?** → `getNetCashAccountBalances`. **Cross-account cash-flow roll-up (spend/income trend)?** → `getNetCashBankActivitySummary`.
- **Cumulative balance over time, cross-table joins, custom-segment groupings?** → `ns_runCustomSuiteQL` (see `suiteql-fallback-patterns.md`).
- **A named saved search?** → run it directly via `ns_listSavedSearches` + `ns_runSavedSearch` (see the SKILL's "Saved searches" workflow); reconstruct in SuiteQL only as a fallback.
- **User said an account name but you're not sure of the internal id?** → Call `getNetCashBankAccounts` first.

When in doubt, run `getNetCashFieldsForTable` on the relevant table to confirm field names rather than guessing — this beats a tool call that fails on a typo.
