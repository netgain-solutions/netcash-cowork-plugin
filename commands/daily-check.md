---
name: daily-check
description: 2-minute morning check-in on NetCash reconciliation status — what changed since last sync, what's outstanding, what needs attention today.
argument-hint: "[account-name]"
---
You are responding to the user invoking /daily-check. The user wants a fast, scannable snapshot of where their bank reconciliation stands this morning. Use the `recon-dashboards` skill in **daily-check mode**. Forward $ARGUMENTS as the account scope (or all accounts if no argument).

Surface only items needing attention this morning — don't dump the full list. Prioritize: bank transactions added since last sync, unmatched items > 7 days old, and accounts with stale reconciliations. Lead with the count and the top 3-5 items by impact; offer to drill in if the user wants more detail.

If multiple bank accounts are connected, group by account. If OneWorld, additionally group by subsidiary.
