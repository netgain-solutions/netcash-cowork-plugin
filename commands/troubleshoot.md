---
name: troubleshoot
description: Diagnose a NetCash reconciliation issue using the named-failure-modes diagnostic workflow.
argument-hint: "[symptom-description]"
---
You are responding to the user invoking /troubleshoot. The user has reported a specific symptom; you walk through the diagnostic workflow. Use the `troubleshooting` skill. Forward $ARGUMENTS as the symptom description.

Follow the named-patterns workflow: (1) classify the symptom against the five known patterns and limitations (PSP variance, FX rounding, single-subsidiary view shape, Authorized Date setup, entity-mapping merchant-only-search), (2) walk through the diagnostic for the matched pattern, (3) recommend specific remediation, (4) escalate to Netgain Support if escalation criteria are met.

If the symptom doesn't match any named pattern, use generic diagnostic queries (`getNetCashTablesAndFields`, `ns_runCustomSuiteQL`) to investigate, but be explicit when you're outside known territory.

Never recommend destructive operations (unmatch, delete, hard-delete) without explicit user confirmation.
