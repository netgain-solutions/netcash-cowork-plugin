# NetCash Cowork Plugin

An AI copilot for NetCash bank reconciliation in NetSuite. Six skills, nine slash commands, designed for controllers and accountants who want help with the daily work.

## What it does

The plugin gives Claude Cowork specialized knowledge for NetCash workflows so accountants can:

- **Find matches** between unmatched bank transactions and GL entries / open invoices / open bills, with confidence-scored suggestions and explicit user confirmation before any match is submitted.
- **Suggest automation rules** based on patterns in manual-match history — GL Match, Create Transaction, Create Transfer, and Cash Application rule types.
- **Generate dashboards and reports** — daily check-ins, cash position across accounts, per-account reconciliation status, and exception summaries. Output stays in conversation by default; Excel/PowerPoint export is available if Anthropic's `xlsx-author` / `pptx-author` skills are also installed in your workspace ([details in CONNECTORS](CONNECTORS.md)).
- **Investigate ad-hoc questions** about bank activity, GL activity, and reconciliations using natural language ("show me last 30 days of unmatched ACH over $10K from Stripe").
- **Surface patterns and anomalies** in bank activity that controllers might miss in line-by-line views — recurring spend, PSP fee patterns, vendor concentration, month-over-month deltas, unusual activity.
- **Troubleshoot recon issues** through a structured diagnostic workflow that names five common patterns and limitations: two expected variance patterns NetCash already handles structurally (PSP fee variance, FX rounding), and three real limitations or setup issues (single-subsidiary view shape, Authorized Date setup for credit cards, entity-mapping merchant-only-search).

## What's not in scope (v1 pilot)

Some NetCash features are outside this plugin's pilot scope because they don't have dedicated MCP tools yet. The plugin will not be able to help directly with:

- **Direct Cash Flow** classification (DCF section / sub-section assignment, classification overrides on transaction lines).
- **Sourcing** records and configuration (the sourcing data model that links bank accounts to FX gain/loss accounts and other defaults).
- **Spending** views and analysis.
- **Activity History** audit log.

If a controller asks about these areas, route them to the relevant NetCash UI directly rather than attempting MCP-driven analysis. Future plugin versions may add MCP coverage for these areas.

## Who it's for

Controllers, senior accountants, and accounting teams at NetSuite-using companies who:

- Already use NetCash for bank reconciliation
- Have Claude Cowork installed (Team or Enterprise plan)
- Want AI assistance with the day-to-day judgment work — investigating breaks, drafting commentary, building automation.

## Prerequisites

Before installing this plugin, the customer must have the **NetSuite MCP server connected** in their Claude Cowork session. The plugin uses NetCash-specific MCP tools (registered within the customer's NetSuite MCP) to read bank activity, GL activity, reconciliations, and to submit matches.

See [CONNECTORS.md](CONNECTORS.md) for the connection walkthrough.

The plugin tells Cowork it needs the NetSuite MCP server. It does not embed a URL or credentials — Cowork manages those per-tenant via the Connectors flow.

## Installation

This plugin installs into Claude Cowork. End users install it in one click from their plugin browser, but a workspace admin needs to enable it for the workspace first via one of two paths: connecting this GitHub repo as a marketplace (recommended, auto-syncs updates) or uploading a ZIP file (no GitHub access required, manual updates).

### Workspace admin (one-time setup)

**Option A: GitHub marketplace (recommended).** Auto-syncs new plugin versions as Netgain ships them.

1. Sign in to Claude Cowork as a workspace admin.
2. Go to **Organization settings → Plugins**.
3. Add a new GitHub plugin marketplace pointing at `netgain-solutions/netcash-cowork-plugin`.
4. Authenticate to GitHub when prompted. Cowork installs Anthropic's GitHub App on the repo for ongoing read access. The repo is public, so no collaborator access is required.
5. Set the install policy. "Available for install" lets users opt in; "Installed by default" enrolls everyone in the workspace.
6. Save.

**Option B: ZIP upload (no GitHub required).** Best if your team can't or doesn't want to use GitHub.

1. Get the plugin ZIP file from your Netgain contact.
2. Sign in to Claude Cowork as a workspace admin.
3. Go to **Organization settings → Plugins**.
4. Use the manual upload flow to upload the ZIP.
5. Set the install policy and save.

Tradeoff: every new plugin version requires re-uploading a fresh ZIP. Your Netgain contact will let you know when updates are available.

For full admin docs, see [Manage Claude Cowork plugins for your organization](https://support.claude.com/en/articles/13837433-manage-claude-cowork-plugins-for-your-organization).

### End users (controllers, accountants)

1. Open the Claude Cowork desktop app.
2. Click **Customize** in the left sidebar (with the Cowork tab selected) to open the plugin browser.
3. Find **NetCash** in the list and click **Install**.
4. Make sure the **NetSuite MCP** is connected in your Cowork session - see [CONNECTORS.md](CONNECTORS.md).
5. The nine slash commands and six skills become available immediately.

For step-by-step end-user install instructions written for non-technical audiences, point your customers at the Netgain KB article on installing the NetCash plugin (your Netgain contact has the link).

### Claude.ai web chat

Claude.ai web chat does not support plugin installation for end users. This plugin is Cowork-only.

## Quick Start

Once your workspace admin has enabled the plugin and you've connected the NetSuite MCP ([Connectors](CONNECTORS.md)), open a new Cowork conversation and try this four-step first session. Total time: about three minutes.

### 1. Verify the connection (15 seconds)

Ask Claude:

> What NetCash tools are available in this session?

You should see roughly 30 NetCash-specific tools, including `getNetCashBankActivity`, `getNetCashReconciliations`, and `submitNetCashManualBankMatch`. If you only see generic NetSuite tools or an empty list, jump to [Connectors — Common issues](CONNECTORS.md#common-issues) before continuing.

### 2. Run `/daily-check` (1 minute)

```
/daily-check
```

The plugin pulls bank activity since your last sync, surfaces unmatched items more than 7 days old, and flags accounts with stale reconciliations. You'll see a count of what changed, the top three to five items by impact, and an offer to drill in. Nothing is submitted or modified — `/daily-check` is read-only.

### 3. Run `/cash-position` (30 seconds)

```
/cash-position
```

Account and GL balances across the bank and credit-card accounts your NetSuite role can see, as of today (or any date you name). On OneWorld, balances are grouped by subsidiary. If your role restricts which accounts are visible, the plugin surfaces only those accounts and tells you so rather than presenting a partial view as complete.

### 4. Ask a question in natural language (1 minute)

Skills auto-trigger on conversation patterns — slash commands are optional. Try one of these against your own data:

> Show me the largest deposits on [your operating account] this month

> What are my biggest expense vendors this quarter?

> Any unusual activity on the [your PSP clearing account] account?

The plugin routes to NetCash MCP tools, pulls the data, and answers with specific transactions, amounts, and dates. If the data doesn't support the question (e.g., the account is empty in that range), it says so rather than making something up.

After this first session, you've used three of the daily-driver commands and seen the natural-language entry point. From here, `/find-matches`, `/suggest-rules`, and `/troubleshoot` are the next commands worth exploring — see [GETTING-STARTED.md](GETTING-STARTED.md) for a longer walkthrough with example outputs, or the [Slash commands](#slash-commands) reference below for the full command list.

## Slash commands

The nine commands give controllers obvious doors into the plugin's capabilities. Skills also auto-trigger on conversation patterns (e.g., asking "why didn't this match?" auto-loads the matching skill), so you don't have to use a slash command every time.

| Command | What it does |
|---|---|
| `/daily-check` | 2-min morning check-in — what changed since last sync, what's outstanding, what needs attention today |
| `/find-matches [account]` | Investigate unmatched items on the named account — runs the six-pass matching workflow |
| `/suggest-rules [account] [days]` | Analyze manual-match history (default 60-90 days) to suggest automation rules |
| `/cash-position [account]` | Account & GL balances across bank and credit-card accounts (as of any date; grouped by subsidiary on OneWorld) |
| `/recon-status [account]` | Per-account reconciliation progress for the current period |
| `/exception-summary [account]` | Top breaks, duplicates, and anomalies sorted by impact |
| `/troubleshoot [symptom]` | Diagnose a specific reported issue using named failure modes |
| `/investigate-variance [item]` | Drill into a specific transaction or variance |
| `/insights [account]` | Surface recurring spend, PSP fee patterns, vendor concentration, MoM deltas, anomalies |

## Skills (auto-triggered)

You don't need to invoke skills explicitly — Claude loads them automatically when the conversation matches their description.

- `matching` — pairing bank transactions to GL transactions / open AR/AP
- `automation-rules` — building rules from manual-match patterns
- `recon-dashboards` — status snapshots and check-ins
- `ad-hoc-reporting` — natural-language data queries
- `pattern-insights` — surfacing trends and anomalies
- `troubleshooting` — diagnosing recon issues

## Interactive NetSuite panels

NetSuite's MCP server returns some responses as interactive panels (searchable transaction lists, filter UIs, report apps) that render inline in your Cowork chat. When that happens, you can interact with the panel directly without going back through Claude — type in the panel's own search box, click filters, change dropdowns, and the panel fetches fresh data from NetSuite on its own. This uses Anthropic's [MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview) capability and works in Cowork and Claude.ai web. On hosts that don't yet support MCP Apps, the same tool calls return plain data instead of the panel — the plugin works either way.

## Tone and behavior

The plugin is built to be a careful, controller-aligned copilot:

- **Always asks for confirmation** before any write operation (submitting a match, creating an automation rule, applying a payment). Never silently writes.
- **Surfaces tool errors** rather than guessing. If a NetCash tool fails, the plugin tells you why and what to do.
- **Cites specific transactions and amounts** rather than general claims. If the data doesn't support an answer, the plugin says so rather than inventing one.
- **Defaults to NetCash-specific MCP tools** over generic NetSuite tools. Falls back to SuiteQL only when canned NetCash tools can't express the request.

## Pilot considerations

A few areas of NetCash have known rough edges that the plugin works around during the v1 pilot. These are tracked and resolve as NetCash MCP coverage expands:

- **Bank-feed integrations and automation rules.** Some bank-feed integrations don't auto-trigger NetCash's automation-rule chain when transactions land. If you've set up rules that aren't firing on transactions from a particular provider, `/troubleshoot` can diagnose whether this is the cause.
- **Single-subsidiary instances.** On non-OneWorld NetSuite, subsidiary-grouped views (cash position by subsidiary, exception summary by subsidiary) collapse to one bucket. The plugin detects this and drops the subsidiary column from output rather than rendering redundant rows — it'll tell you when it's doing so.
- **Role-based account restrictions.** If your NetSuite role restricts which bank accounts you can see, NetCash enforces this server-side. The plugin surfaces only the accounts your role can access and says so explicitly rather than presenting a partial view as complete.

## Versioning

Semantic versioning. `0.1.x` = internal pilot. `0.x.x` = pilot phase. `1.0.0` = public Netgain release. See [CHANGELOG.md](CHANGELOG.md) for what changed in each version.

## License

Apache License 2.0. See [LICENSE](LICENSE).

## Support

**Pilot users:** Contact your Netgain pilot lead with feedback, questions, or issues. Bug reports filed in this GitHub repository are also welcome.

**Setup issues** (NetSuite MCP won't connect, NetCash tools not visible, authentication errors): see [CONNECTORS.md — Common issues](CONNECTORS.md#common-issues) first.

**Plugin issues** (wrong output, unexpected behavior, skill errors): try `/troubleshoot` with a description of the symptom — the plugin has a named-failure-mode diagnostic that resolves five common patterns directly. If `/troubleshoot` can't diagnose, file a GitHub issue or escalate to your Netgain pilot lead.

For Netgain Support contact information, see netgain.tech.
