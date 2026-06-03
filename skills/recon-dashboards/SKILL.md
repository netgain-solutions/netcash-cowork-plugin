---
name: recon-dashboards
description: Show a controller a daily, weekly, or monthly snapshot of NetCash reconciliation status — current cash position across all accounts, per-account match progress, top exception items, and what needs attention this morning. Use this skill when the user asks for cash position, what's outstanding, where they are in the close, recon status, or what needs attention. Also use proactively at session start if the user mentions starting their reconciliation work for the day.
argument-hint: "[account-name]"
---

## Purpose

This skill is built for the daily 2-minute check-in, not the deep monthly dive. Modern bank-reconciliation guidance has shifted: high-volume cash accounts should be reconciled daily or near-daily, not month-end-only, because the cost of finding a break a day after it happens is dramatically lower than finding it 30 days later. Controllers often spend a meaningful share of their reconciliation time pulling and reformatting data; a fast, scannable dashboard collapses that gathering into a single tool turn.

The skill answers four questions, in four view modes:

- **Daily check** — what changed since I last looked, what's stuck, what's stale?
- **Cash position** — live Account Balance, GL Balance, and variance per account (bank **and** credit card; per subsidiary on OneWorld), as of today or any date you name. Flags accounts needing attention by recon status.
- **Recon status** — where are we in the close — which accounts are done, in-progress, not started?
- **Exception summary** — what are the top breaks, sorted by what'll hurt most if left unhandled?

It does not propose remediations. Diagnosing why something is broken and how to fix it belongs to `troubleshooting`. If a dashboard view surfaces something that needs action, suggest the relevant slash command (`/find-matches`, `/troubleshoot`, `/investigate-variance`) and stop. The user drives the next step.

## Workflow
Detect which view the user is asking for from the active slash command, the phrasing of their request, or `$ARGUMENTS`. The four view modes are distinct enough that running the wrong one wastes both a tool turn and the user's attention.

- **Slash command routing.** If invoked via `/daily-check`, `/cash-position`, `/recon-status`, or `/exception-summary`, use that view — these commands are unambiguous.
- **Clear phrasing routing.** If the request maps cleanly to one view ("show me cash" → cash position; "where are we in the close" → recon status; "biggest exceptions" → exception summary; "what changed overnight" → daily check), use it.
- **No view specified (skill invoked directly, or ambiguous phrasing).** When the user runs `/netcash:recon-dashboards` with no view in `$ARGUMENTS`, or asks something generic like "what's going on?", "give me a summary", "where do I stand?", **present the four views with one-line descriptions before doing anything else**. The user shouldn't have to know the view names to pick. Use this exact picker shape (descriptions written so a controller who's never seen the views can pick correctly):

  > Which dashboard view?
  > - **Daily check** — what changed since you last looked: new bank activity, items aging past 7 days, recons stuck for 5+ business days. The 2-minute morning check-in.
  > - **Cash position** — live Account Balance, GL Balance, and variance per account (bank and credit card; per subsidiary on OneWorld), as of today or a date you name. Flags accounts needing attention by recon status.
  > - **Recon status** — where each account stands in the current close: not-started, in-progress, stuck, complete. Shows what's blocking the close.
  > - **Exception summary** — top unmatched bank transactions for the open period plus prior 30 days, sorted by absolute amount (biggest financial impact first).

  Then stop. Do not call any tool, do not pre-select an option, and **do not guess based on prior turns in the session** — even if an earlier response suggested one of these views as a follow-up, the user may have changed direction. Wait for the user's pick.

Confirm the chosen view in one line at the top of the response so the user knows what frame they got. **Lead that line with the word "Reconciliation"** — e.g., *"Reconciliation dashboard — exception summary view"* or *"Reconciliation dashboard — cash position"*. This disambiguates the abbreviation "recon" (in the slash command and skill name) from "reconnaissance" for chat-titling models, which otherwise tend to name the conversation "Reconnaissance ..." instead of "Reconciliation ...".

If `$ARGUMENTS` includes an account name, scope the view to that account. Otherwise, scope to all active accounts (call `getNetCashBankAccounts` first to enumerate).

**Daily check.**

1. Call `getNetCashBankActivity` for the last 24 hours of activity (use `lastUpdatedDate` as the filter, not `transactionDate` — sync timing matters more than transaction date for this view). Bucket new vs. previously-known.
2. Call `getNetCashBankActivity(isOnlyUnmatched=true)` filtered to `transactionDate` more than 7 days old. These are the aging items.
3. Call `getNetCashReconciliations` for the current period. Identify in-progress or not-started recons that haven't moved in 5+ business days.
4. Render the daily check template (see `references/dashboard-templates.md`).

**Cash position.**

1. Call `getNetCashAccountBalances` for the balances. Defaults to today; pass `asOfDate` (YYYY-MM-DD) if the user names a date. It returns, per bank **and** credit-card account, `accountBalance` (Account Balance), `glBalance` (GL Balance), and `variance` (signed GL − Account) — the same numbers the NetCash dashboard balance widget shows. Use these directly; never compute, derive, or sum your way to a balance.
2. Call `getNetCashReconciliations` with `columnSet='detailed'` for the same accounts to get recon **status** — the "needs attention" signal — and, on OneWorld, the subsidiary each account rolls up to. (Use `detailed`: it returns both `status` and `subsidiary`; the `financial` set omits `status`, and `default` omits `subsidiary`.) Join to the balances on account ID. You are NOT using the recon record for balances anymore; only status (and subsidiary).
3. **Do not** call `getNetCashBankActivitySummary` for balances (see Hard Constraints below). It returns transaction-flow stats with a sign convention that differs between bank accounts and credit cards — a cash-flow trend tool, not a balance tool.
4. Multi-currency / OneWorld base currency: pass `consolidatedSubsidiary` (the subsidiary internal ID) to `getNetCashAccountBalances` to get `baseAccountBalance` / `baseGlBalance` alongside the native-currency balances.
5. Render the cash position template (see `references/dashboard-templates.md` Template 2). Credit-card accounts are now real rows — show bank cash and credit-card balances in **separate** totals; never sum a credit-card balance (a liability) into bank cash. When several cards share one GL account (common with Brex/Ramp — each cardholder is a separate account row rolling up to one GL liability account), every card row repeats that one GL balance: sum **Account Balance** per cardholder, but show the **GL balance once per GL account** — never sum the repeated GL across cardholder rows (it would multiply the liability by the number of cards).
6. **Label the data source explicitly.** Lead the view with the as-of date — *"Balances as of <date>"* — so the user knows the point-in-time. These are live as-of-date balances that mirror the NetCash dashboard balance widget, not a recon-close snapshot.

**Recon status.**

1. Call `getNetCashReconciliations` filtered to the current period.
2. Bucket by status: `not-started`, `in-progress`, `reopened`, `complete`.
3. Optionally enrich the in-progress entries by calling `getNetCashReportMatches` for match counts (works on all instances; subsidiary breakdown only renders on OneWorld — see OneWorld Handling below).
4. Render the recon status template.

**Exception summary.**

1. Call `getNetCashBankActivity(isOnlyUnmatched=true)` for the relevant scope. Default lookback is the current open period plus the prior 30 days. Don't paginate beyond what the tool returns in one call — these are exceptions, not the full ledger.
2. Sort by absolute amount descending, then by `transactionDate` ascending as the tiebreaker (older items get priority within the same amount).
3. Render the exception summary template.

For all four views, surface staleness explicitly when the data is stale (see Hard Constraints below — never show stale data without flagging it). If a sync is more than 24 hours old or a recon snapshot is older than the user's expected refresh cadence, say so at the top of the view, in plain language.

## OneWorld Handling
Some recon-dashboards views show subsidiary breakdowns that are only meaningful on OneWorld. Detect OneWorld before deciding the output shape.

**Detection rule** (canonical, defined in plugin-level `CLAUDE.md`): call `ns_getSubsidiaries` and ignore consolidated entries (negative IDs) — more than one real subsidiary → OneWorld; zero or one → single-subsidiary. If `ns_getSubsidiaries` errors or is ambiguous, fall back to `ns_runCustomSuiteQL` with `SELECT COUNT(*) AS sub_count FROM subsidiary` (`sub_count > 1` → OneWorld; `sub_count = 1`, or a "table not found" error, → single-subsidiary).

**OneWorld branches.**

- `getNetCashReportMatches` enrichment for the recon-status view shows match-count progress per recon broken out by subsidiary (e.g., "Sub A: 142/167 matched, 85%"). Use it for the exception-summary view if the user wants per-recon match-rate context.
- The cash-position view groups by subsidiary in addition to by account.

**Non-OneWorld branches.**

- `getNetCashReportMatches` still runs and still returns match-count data; the subsidiary column is the part that isn't meaningful on a single-subsidiary instance. Use it for match-progress enrichment without the subsidiary breakdown.
- The cash-position view groups by account only (no subsidiary grouping). Drop the subsidiary column from the templates.
- When you drop a subsidiary-grouped view because the customer is on a single-subsidiary instance, mention it once at the bottom of the response. Do not silently degrade — the user should know that the subsidiary breakdown isn't applicable.

If OneWorld detection fails on both paths (e.g., `ns_getSubsidiaries` errors AND the `ns_runCustomSuiteQL` fallback count returns an authentication error or is unavailable for non-table-not-found reasons), surface the error and stop. Do not guess; do not proceed with the OneWorld branch as a default. A "table not found" error on the fallback count is specifically a positive signal that the instance is single-subsidiary — handle it as such, don't surface it as an error.

## Hard Constraints
NEVER render a subsidiary-grouped view (cash position by subsidiary, recon status by subsidiary) without first detecting OneWorld per the rule above (`ns_getSubsidiaries`, falling back to a `ns_runCustomSuiteQL` subsidiary count). On a single-subsidiary instance, subsidiary grouping is redundant and confusing — drop the subsidiary column or section and surface that you did so.

NEVER compute or derive balances from `getNetCashBankActivitySummary`. The tool returns sums of debits/credits with a sign convention that differs between bank accounts (`total_amount` = debit − credit) and credit cards (`total_amount` = credit − debit), and the `net_meaning` interpretation block has been observed to contradict the underlying math. Any agent that tries to compute a balance from this tool ends up double-counting prior-period activity into the cutover initial balance and producing numbers 5-10× too high. Use `getNetCashAccountBalances` for balance reporting — it returns canonical `accountBalance`, `glBalance`, and `variance` per bank and credit-card account, as of any date, mirroring the NetCash dashboard balance widget. The activity summary is for cash-flow *trend* analysis (was this period higher or lower than last?), not balance reporting.

NEVER show stale data without flagging staleness. If the most recent `lastUpdatedDate` on bank activity is more than 24 hours old, or if a reconciliation snapshot is older than the user's expected refresh cadence, say so at the top of the view in plain language. Stale numbers shown without a staleness label are worse than no numbers at all — the user trusts them and acts on them. **Sandbox exception:** before raising a staleness alarm, check the `isSandbox` flag on the MCP response (NetCash returns it on every tool response). On a sandbox/test instance the data is often a frozen dataset that simply ends at a fixed historical date, so a "sync is X months stale" warning implies a production sync gap that doesn't exist — reframe it as *"test dataset, last activity <date>"* instead. Don't suppress it entirely; the user still wants to know the data isn't live, just framed correctly.

NEVER recommend remediation in this skill. Dashboards surface what's wrong; they don't fix it. If the user wants to know what to do about an exception, route to `troubleshooting`. If they want to investigate a specific transaction, route to `ad-hoc-reporting` (`/investigate-variance`). If they want to find a match, route to `matching` (`/find-matches`). The dashboard's last line should suggest the relevant follow-on slash command, not walk through diagnosis.

NEVER treat `balance_difference != 0` as evidence of a current recon variance. The `balance_difference` field on a recon record is the tautological subtraction `gl_balance − bank_balance` — it's non-zero whenever there are outstanding items in flight (DIT, outstanding checks), which is normal for any active account. The correct variance signal is **recon status** (`BALANCE_CHANGED`, `STUCK`, `NOT_STARTED`) and **Open Bank Activity** items (bank-side transactions without a GL counterpart). For the full framework, the meaning of each field, and the worked example that motivated this rule, see `references/recon-interpretation.md`.

## Output Format
The default output is in-conversation: concise, scannable tables, sorted so the most action-needing item is first.

**Layout discipline.**

- Lead with a one-sentence headline that summarizes the view ("3 new transactions overnight, 8 outstanding > 7 days, 1 stale recon").
- Use tables for any data with three or more comparable items. Reserve prose for what the data means or what the user should do next.
- Sort every table by what-needs-attention-most. For exceptions, that's absolute amount descending. For aging items, that's days-outstanding descending. For recon status, that's "stuck" before "in-progress" before "not-started" before "complete".
- Truncate long lists at 10-20 items. If there are more, note the total count and suggest the slash command for the full list (e.g., "showing top 10 of 47 — run `/exception-summary` for the full list").
- End with a one-line suggested next step — a slash command if there's an obvious follow-up, otherwise nothing. Do not pad the end with restating the headline.

The four templates (daily-check, cash-position, recon-status, exception-summary) are documented in `references/dashboard-templates.md`. Follow them; don't improvise on the section structure or the sort order. They've been calibrated for what controllers actually scan in two minutes.

**Workbook output (Excel).**

If the user asks for a workbook, an Excel file, "send me a download", or any explicitly file-based output format, route to Anthropic's pre-built `xlsx-author` skill (or `anthropic-skills:xlsx`). Pass the table data structured by view (one sheet per view if multi-view, otherwise one sheet for the requested view). This skill produces in-conversation output by default; the moment the user signals they want a file, hand off to the workbook-authoring skill rather than reformatting in markdown.

If the user asks for a PowerPoint instead, defer to `anthropic-skills:pptx` with the same table data. Same principle: this skill is the in-conversation dashboard; specialized formatters own file output.
