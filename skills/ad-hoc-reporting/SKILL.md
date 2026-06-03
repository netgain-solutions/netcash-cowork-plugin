---
name: ad-hoc-reporting
description: Answer specific ad-hoc questions about bank activity, GL activity, and reconciliations using NetCash data — natural-language queries like "show me last 30 days of unmatched ACH over $10K from Stripe", "biggest expense vendor this month", "compare deposits this month vs last", or "list all transactions on BOA in November". Also use to investigate or drill into a specific transaction or recon variance. Prefer canned NetCash tools (`getNetCashBankActivityByGrouping`, `getNetCashBankActivity`) over ad-hoc SuiteQL.
argument-hint: "[query]"
---

## Purpose

Bank-reconciliation work commonly involves significant time pulling lists, filtering Excel, joining bank activity to GL, and comparing month-over-month — overhead that varies widely by company but is rarely zero. This skill exists to compress that overhead. The user types a natural-language question; you return the answer.

The point is to replace the manual NetSuite saved-search → CSV export → Excel pivot loop with a single conversational round-trip. Every query the user would otherwise reach for Excel to answer should be answerable here, in the chat, with cited data from NetCash.

## Workflow
For every user query, run this sequence:

1. **Parse the question.** Identify the entities: account(s), date range, transaction type (ACH / wire / check / deposit / etc.), amount filter, vendor or merchant, matched/unmatched status, subsidiary, currency. If any of these are ambiguous and matter for the answer, ask one focused clarifying question — do not guess.

2. **Decide canned-tool vs. SuiteQL.** Default to a canned NetCash tool. Only escalate to `ns_runCustomSuiteQL` when no canned tool can express the filter (cross-table joins, time-series cumulative balances, custom-segment groupings — see `references/suiteql-fallback-patterns.md`).

3. **Map question → tool + parameters.** Use the preference order in Query Mapping Principles below. If the user asked for an aggregate ("biggest", "total", "by vendor", "by month") prefer `getNetCashBankActivityByGrouping`. If they asked for a list, prefer `getNetCashBankActivity` with the right `columnSet` mode. See `references/query-patterns.md` for ~20 worked examples.

4. **Call the tool.** Pass exact filters. Do not call multiple tools in parallel for the same question unless the question is genuinely two queries (e.g., month-over-month comparison).

5. **Format the result** per Output Format below. If the result has zero rows, say so explicitly — do not paraphrase as "no significant activity" or "nothing notable".

6. **Suggest a follow-on slash command** when the result naturally points to one. Examples: an unmatched-ACH list points to `/find-matches`; a recurring vendor with manual matches points to `/suggest-rules`; an unexplained variance points to `/troubleshoot`.

If the user asks for the result as a workbook, Excel file, or PowerPoint, hand off to Anthropic's pre-built `xlsx-author` skill with the result data — see Output Format below.

## Recon variance investigations

When the user's query is about a reconciliation variance — e.g., *"investigate the variance on my [Account] April recon"*, *"why is this recon off by $4,000?"*, *"what's wrong with the BOA recon"* — follow this gating sequence **before** pulling data on causes. `balance_difference != 0` on a recon record is the tautological `gl_balance − bank_balance` subtraction and is non-zero whenever outstanding items exist; it is not a variance signal by itself.

1. **Pull the recon via `getNetCashReconciliations` and read `status` first.** If status is `COMPLETED` or `AUTO_RECONCILED`, the user has signed off — confirm with them what they're seeing as a variance before chasing one. If status is `BALANCE_CHANGED`, the underlying transactions changed since submission and the user needs to reopen the recon to acknowledge them (this is not a current variance signal by itself).

2. **Check Open Bank Activity for the period.** Unmatched bank-side items via `getNetCashBankActivity` with `isOnlyUnmatched: true` filtered to the account and period. These are the real variance candidates. If empty, there's no current bank-side variance regardless of what `balance_difference` says.

3. **Only after steps 1–2 do you have grounds to chase a variance cause.** If the user's concern is just "this number isn't zero", they're often pointing at outstanding items in flight (DIT, outstanding checks) — surface that framing rather than going deeper.

The full recon-interpretation framework — the four-category line classification, what each field means, what `Balance Changed` status actually signals — lives in [`skills/recon-dashboards/references/recon-interpretation.md`](../recon-dashboards/references/recon-interpretation.md). Read it before any recon-related variance investigation.

## Query Mapping Principles
Tool preference order (always try in this order, escalate only when the previous tool can't express the question):

1. **`getNetCashBankActivityByGrouping`** — for any aggregation question. "Biggest", "total", "sum by", "count of", "top N by amount", "by vendor / by month / by category / by subsidiary". This tool handles `groupBy` natively; do not aggregate manually in the model when this tool can do it.

2. **`getNetCashBankActivity`** — for filtered lists of bank transactions. Five `columnSet` modes (full reference in `references/query-patterns.md`):
   - `minimal` — id, date, amount, description. Use for quick lookups.
   - `default` — adds match status, account, type. Use when user wants a normal list view.
   - `detailed` — adds entity, memo, transaction-type detail. Use for investigation.
   - `financial` — adds currency, exchange rate, FX-related fields. Use for multi-currency questions.
   - `all` — every available column. Use when the user explicitly asks for "everything" or you don't know yet what fields they need.

3. **`getNetCashGLActivity`** — when the question is about the GL side (journal entries, GL posting account, GL period). Mirrors `getNetCashBankActivity`'s filtering shape.

4. **`getNetCashCashApplicationTransactions`** — when the question is about open AR or open AP (unapplied invoices, unpaid bills). Use when the user asks "what's outstanding for [customer]" or "open bills for [vendor]".

Beyond these four, fall back to:
- `getNetCashAccountBalances` — for cash position / account & GL balances across accounts (live, as of any date; bank and credit card). The canonical balance tool — don't derive balances from other tools.
- `getNetCashReconciliations` — for recon-status questions.
- `getNetCashBankActivitySummary` — for cash-**flow** trend roll-ups (period spend/income), NOT balances.
- `ns_runCustomSuiteQL` — last resort, only for queries the canned tools cannot express. See `references/suiteql-fallback-patterns.md`.

When you escalate to SuiteQL, briefly state why a canned tool couldn't handle the question (one sentence). This is a deliberate exception, not the default.

## Hard Constraints
NEVER fabricate data, vendor names, transaction details, or SQL results. If a tool returns no rows, say "no rows returned for that filter" — do not invent example data, do not extrapolate from training knowledge, do not "fill in" a plausible-looking transaction. Every number, name, date, and amount you present must come from a tool call in this conversation.

NEVER show approximate numbers without an explicit "(approximate)" label. If you rounded, say so. If a number is derived (e.g., trailing-12-month average), say how you derived it. If a tool returned a `count` field that's a hint rather than a precise total, label it.

Prefer canned NetCash tools (`getNetCashBankActivity`, `getNetCashBankActivityByGrouping`) over ad-hoc SuiteQL whenever a canned tool can express the question. SuiteQL is the fallback for queries the canned tools cannot express (cross-table joins, time-series cumulative balances, custom-segment groupings) — not a shortcut for questions you don't feel like mapping. When you do reach for `ns_runCustomSuiteQL`, write one sentence explaining why no canned tool worked.

## Output Format
Match the format to the question shape:

- **Aggregations / groupings** — table with the grouping column on the left and the metric column on the right. Sort by the metric descending unless the user specified another order. Include a "Total" row at the bottom for sum-style aggregates. Do not include columns the user didn't ask about.

- **Single-answer questions** ("biggest deposit last week", "what was the total ACH volume on BOA in November") — answer in one sentence with the specific number, then one line of supporting context (the underlying transaction or filter). Do not pad with a multi-row table when the user asked one question.

- **Lists** — table with the columns the question implies (date, amount, description, entity, account). Use `columnSet` to keep the table focused. If the result has more than ~20 rows, show the top 20 (sorted by what the user cares about — usually amount or date) and tell the user the total count.

- **Comparisons** ("this month vs last", "by-account YoY") — side-by-side table with delta and delta-% columns. Lead with one sentence summarizing the headline change.

- **Empty results** — say "no rows returned for [filter]" explicitly, then offer a way to broaden (e.g., extend the date range, drop the amount threshold).

After the result, suggest a single follow-on slash command when one is obvious. Examples:
- A list of unmatched bank transactions → suggest `/find-matches`.
- A list of repeat manual matches by vendor → suggest `/suggest-rules`.
- An unexplained variance or anomalous amount → suggest `/troubleshoot`.
- A pattern that warrants deeper analysis → mention that the `pattern-insights` skill can drill in.

If the user asks for a workbook, Excel file, spreadsheet, ".xlsx", PowerPoint, or "send me a file", hand off to Anthropic's pre-built `xlsx-author` skill with the result data already in hand. The dashboard skill itself produces in-conversation tables by default; only escalate to file output on explicit request.

## Tool Routing
Primary NetCash tools for this skill:
- `getNetCashBankActivity` — filtered bank transaction lists.
- `getNetCashBankActivityByGrouping` — aggregates / group-bys.
- `getNetCashAccountBalances` — cash position / account & GL balances (live, as of any date; bank and credit card).
- `getNetCashBankActivitySummary` — cross-account cash-**flow** trend roll-ups (not balances).
- `getNetCashGLActivity` — GL-side queries.
- `getNetCashCashApplicationTransactions` — open AR / open AP.
- `getNetCashReconciliations` — recon-status queries.
- `getNetCashBankAccounts` — to enumerate accounts when the user names one ambiguously.

Schema discovery (when you don't know what filter field to pass):
- `getNetCashTables` — list available tables.
- `getNetCashTablesAndFields` — table + field listing.
- `getNetCashFieldsForTable` — fields for a specific table.

Fallback (only when canned tools cannot express the question):
- `ns_runCustomSuiteQL` — read-only SuiteQL.

Saved searches (when the user names one — see the workflow below):
- `ns_listSavedSearches` — find a saved search by name (`query` filter).
- `ns_runSavedSearch` — run it by `searchId`, paged with `range_start` / `range_end`.

**Saved searches.** When the user references a NetSuite saved search by name ("run my 'Unmatched Stripe deposits' search", "what saved searches do I have for AR?"), run it directly — don't reconstruct it in SuiteQL:

1. `ns_listSavedSearches` with the `query` param set to the name (or a distinctive word from it) to find candidates. The catalog is NetSuite-wide and large (it includes NetAsset / NetClose / other-product searches), so always filter by name — never enumerate the whole list.
2. One clear match → confirm it before running ("Run 'Unmatched Stripe deposits' (`customsearch_…`)?"). Multiple matches → show the short candidate list (title + id) and let the user pick. No match → say so, then fall back to SuiteQL reconstruction (see `references/suiteql-fallback-patterns.md` Template 4).
3. `ns_runSavedSearch` with the chosen `searchId`. Start with a bounded window (e.g., `range_start: 0`, `range_end: 20`); show the top ~20 and note if the window came back full (more rows likely), paging further with `range_start` / `range_end` only if the user wants them — the tool returns no total-count field, so don't claim a precise total. Pass the `type` param only for the rare standalone search types the tool flags.
4. Render results in the normal table format; offer a follow-on slash command as usual.

Saved searches are read-only (they're queries) — no write confirmation needed, but confirm *which* search before running so you don't fire a heavy or wrong one.

See the plugin-level CLAUDE.md `<tool_routing_principle>` for the universal rule.
