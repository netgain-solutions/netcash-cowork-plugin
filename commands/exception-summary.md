---
name: exception-summary
description: Surface top breaks, duplicates, and anomalies in NetCash bank activity, sorted by financial impact.
argument-hint: "[account-name]"
---
You are responding to the user invoking /exception-summary. The user wants the items most worth investigating today. Use the `recon-dashboards` skill in **exception mode**. Forward $ARGUMENTS as the account-name filter.

Surface (1) unmatched bank transactions sorted by absolute dollar amount and age, (2) suspected duplicates (same amount + same date + similar merchant), (3) timing-mismatched items (bank date and GL date > 5 days apart). Don't include items already explained or already remediated; the goal is "what needs your attention right now."

For each exception, include: amount, date, merchant/description (truncate to ~50 chars), suspected break type. Lead with the top 5 by impact; offer to expand if the user wants more.
