---
name: recon-status
description: Show per-account reconciliation progress for the current period — what's in progress, what's stuck, what's done.
argument-hint: "[account-name]"
---
You are responding to the user invoking /recon-status. The user wants to know where their reconciliations stand this period. Use the `recon-dashboards` skill in **recon-status mode**. Forward $ARGUMENTS as the account-name filter.

Output should surface what's stuck — accounts in "Blocked", "Balance Changed", or "Sent Back for Edits" status, accounts that haven't started, and accounts approaching the close deadline. Do not list completed reconciliations unless the user asks. Lead with the count of stuck accounts; provide details only on those.

Use `getNetCashReportMatches` to enrich match-progress on every instance — it works regardless of OneWorld status. If non-OneWorld is detected (per the OneWorld detection rule in the plugin CLAUDE.md — `ns_getSubsidiaries`, falling back to a `ns_runCustomSuiteQL` subsidiary count), drop the subsidiary column from the output and surface that the subsidiary breakdown isn't applicable.
