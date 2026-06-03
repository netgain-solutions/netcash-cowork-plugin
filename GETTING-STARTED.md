# Getting started with NetCash for Cowork

This walkthrough gets you through your first NetCash session — about 10 minutes, with example outputs and "what if it doesn't work" branches. For a 3-minute version without example outputs, see the [README Quick Start](README.md#quick-start).

## Before you start

You'll need:

- Claude Cowork (Team or Enterprise plan)
- The NetCash plugin installed — your workspace admin enables it once (see [README Installation](README.md#installation))
- The NetSuite MCP connected in your Cowork session (see [CONNECTORS.md](CONNECTORS.md))
- A NetSuite role with NetCash MCP access — start from the **NetCash MCP View** template (read-only) or **NetCash MCP Edit** template (read + write), then have your NetSuite admin clone and customize

Open a new Cowork conversation and work through the four steps below.

## 1. Verify NetCash tools are visible

Ask Claude:

> What NetCash tools are available in this session?

**Example output (illustrative):**

> I can see ~30 NetCash-specific tools in this session, including:
>
> - `getNetCashBankActivity`, `getNetCashBankActivityByGrouping`, `getNetCashBankActivitySummary`, `getNetCashAccountBalances`
> - `getNetCashReconciliations`, `getNetCashReportMatches`
> - `getNetCashAutomationRules`, `createNetCashGlMatchRule`, `createNetCashCashApplicationRule`, ...
> - `submitNetCashManualBankMatch`, `submitNetCashManualCashApplication`
> - `getNetCashGLActivity`, `getNetCashCashApplicationTransactions`
> - `findNetCashEntityMappingsByAlias`, `getNetSuiteEntities`
> - `ns_runCustomSuiteQL` (general-purpose NetSuite SuiteQL fallback)
>
> What would you like to do?

**If you only see generic NetSuite tools** (no `getNetCashBankActivity` etc.): your NetSuite role probably doesn't have access to NetCash records. Ask your NetSuite admin to assign you a role based on the **NetCash MCP View** template (read-only) or **NetCash MCP Edit** template (read + write). See [CONNECTORS — No NetCash tools visible](CONNECTORS.md#no-netcash-tools-visible).

**If you see no tools at all:** the NetSuite MCP isn't connected. Go to **Settings → Connectors** in Cowork and connect the NetSuite MCP. See [CONNECTORS — Connecting the NetSuite MCP server](CONNECTORS.md#connecting-the-netsuite-mcp-server).

## 2. Run /daily-check

Type:

```
/daily-check
```

`/daily-check` pulls bank activity since your last sync, surfaces unmatched items more than 7 days old, and flags accounts with stale reconciliations. It's read-only — nothing is submitted or modified.

**Example output (illustrative):**

> **Morning check, May 20**
>
> Since your last sync (May 19, 4:47 PM):
> - 23 new bank transactions across 4 accounts; 18 auto-matched, 5 unmatched
> - 2 accounts have unmatched items more than 7 days old
> - 1 account flagged for stale feed
>
> **Needs attention today:**
>
> | Account | What | Amount | Why |
> |---|---|---|---|
> | Operating | Unmatched Stripe deposit | $42,118.50 | No GL entry within ±2d, ±$5 |
> | Payroll | Outstanding check #1452 | $3,890.00 | Issued April 30, still uncleared |
> | Subsidiary cash | Stale feed | — | No bank activity ingested since 2025-11-15 |
>
> Want to drill into any of these? Or run `/find-matches` to work through the unmatched items?

**If `/daily-check` returns empty:** either there's no new bank activity since your last sync (good!), or your role can't see any bank accounts. Try `/cash-position` next — it surfaces account visibility explicitly.

**If `/daily-check` times out or errors:** the NetSuite MCP may have hit a session auth timeout. Try once more. If it fails repeatedly, see [CONNECTORS — Common issues](CONNECTORS.md#common-issues).

## 3. Run /cash-position

Type:

```
/cash-position
```

Account and GL balances across the bank and credit-card accounts your NetSuite role can see, as of today (or any date you name). On OneWorld, balances are grouped by subsidiary.

**Example output (illustrative — OneWorld):**

> **Cash position — balances as of May 20**
>
> **Parent**
>
> | Account | Account Balance | GL Balance | Variance (GL − Account) | Recon status |
> |---|---|---|---|---|
> | Operating (BOA) | $1,248,392.50 | $1,235,952.50 | ($12,440.00) | Complete (through April) |
> | Payroll (Chase) | $182,005.00 | $178,115.00 | ($3,890.00) | Complete (through April) |
> | **Total bank cash** | **$1,430,397.50** | **$1,414,067.50** | | |
>
> **Subsidiary A**
>
> | Account | Account Balance | GL Balance | Variance (GL − Account) | Recon status |
> |---|---|---|---|---|
> | Operating (Wells) | $84,716.30 | $84,716.30 | $0.00 | In progress (May) |
>
> **Credit cards (all subsidiaries)**
>
> | Account | Balance | GL Balance | Variance | Recon status |
> |---|---|---|---|---|
> | Brex | $52,300.00 | $52,300.00 | $0.00 | — |
>
> Variance (GL − Account) is the net of outstanding items in flight (checks not yet cleared, deposits in transit) — non-zero is normal, not a problem. Credit-card balances are shown separately and never added into bank cash.

On a single-subsidiary NetSuite tenant, the plugin drops the subsidiary grouping and presents one table — it'll tell you when it does so rather than rendering redundant rows.

**If your role restricts which bank accounts you can see:** the plugin surfaces only those accounts and says so explicitly. You won't see a silent partial view.

## 4. Ask a question in natural language

Skills auto-trigger on conversation patterns — slash commands are optional. Try one of these against your own data:

> Show me the largest deposits on [your operating account] this month

> What were my biggest expense vendors last quarter?

> Compare deposits this month vs last month

The plugin routes to NetCash MCP tools, pulls the data, and answers with specific transactions, amounts, and dates. If the data doesn't support the question (e.g., the account is empty in that range), it says so rather than making something up.

**Some prompts that work less well:**

- *"Show me everything on the bank"* — too broad; the plugin will ask you to narrow before pulling data
- *"How does NetCash work?"* — the plugin is built to work *with* your data, not to explain itself
- *"Reconcile my account"* — reconciliation is a multi-step workflow, not a single command; start with `/recon-status` to see where you are

## Next: the daily-driver commands

Once you're comfortable with the four steps above, three commands cover most of the daily work:

- **`/find-matches [account]`** — Six-pass match workflow on unmatched bank transactions. Runs in batch mode when there are 15 or fewer unmatched (one consolidated bucketed view: Clean / Probably right / Risk-flagged / Special handling) or pass-by-pass mode for larger batches. Writes only on explicit confirmation; partial submits are supported.
- **`/suggest-rules [account] [days]`** — Analyzes recent manual-match history (default 60-90 days) to propose automation rules. Four rule types: GL Match, Create Transaction, Create Transfer, and Cash Application. Always confirms before creating any rule.
- **`/troubleshoot [symptom]`** — Diagnoses recon issues using a named-failure-mode workflow. Five common patterns currently covered: PSP fee variance, FX rounding, single-subsidiary view shape, Authorized Date setup for credit cards, and entity-mapping merchant-only-search.

Four more commands round out the surface:

- `/recon-status [account]` — Per-account reconciliation progress for the current period
- `/exception-summary [account]` — Top breaks, duplicates, anomalies sorted by impact
- `/investigate-variance [item]` — Drill into a specific transaction or variance
- `/insights [account]` — Surface recurring spend, PSP fee patterns, vendor concentration, anomalies

See the [README Slash commands](README.md#slash-commands) table for the full reference.

## A few things to know during pilot

A few areas of NetCash have known rough edges that the plugin works around in v1. The full list is in [README — Pilot considerations](README.md#pilot-considerations). The one most likely to surface in your first sessions:

- **Some bank-feed integrations don't auto-trigger automation rules.** If you've set up rules that aren't firing on transactions from a particular provider, `/troubleshoot` can diagnose whether this is the cause.

## Feedback and support

- **Pilot questions and feedback:** contact your Netgain pilot lead.
- **Setup issues** (MCP won't connect, tools not visible): see [CONNECTORS — Common issues](CONNECTORS.md#common-issues).
- **Plugin issues** (wrong output, unexpected behavior): try `/troubleshoot` first, then file a GitHub issue or escalate to your pilot lead.
