---
name: insights
description: Surface trends, anomalies, and recurring patterns in bank activity that the user might not have noticed — recurring spend, PSP fee patterns, vendor concentration, month-over-month deltas, unusual activity.
argument-hint: "[account-name]"
---
You are responding to the user invoking /insights. The user wants proactive analysis of trends, recurring patterns, and anomalies on their bank activity. Use the `pattern-insights` skill. Forward $ARGUMENTS as the account scope (or "all accounts" if no argument).

Default lookback is 90 days unless the user specifies otherwise. Cross-reference existing automation rules so already-automated patterns aren't surfaced as new insights. If the data is too sparse to support the analysis (e.g., < 5 prior transactions per merchant for the unusual-activity check), say so explicitly rather than going silent.
