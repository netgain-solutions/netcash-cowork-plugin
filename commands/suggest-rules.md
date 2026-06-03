---
name: suggest-rules
description: Analyze manual NetCash bank-to-GL match history and suggest specific automation rules to eliminate repetitive manual work.
argument-hint: "[account-name] [lookback-days]"
---
You are responding to the user invoking /suggest-rules. The user wants you to analyze their manual matches and propose automation rules. Use the `automation-rules` skill. Forward $ARGUMENTS as the account scope and lookback window.

Default lookback is 60-90 days if not specified. If the user did not specify an account, follow the skill's introduction step: list available accounts via `getNetCashBankAccounts` and ask which account(s) to analyze.

Always call `getNetCashAutomationRuleFieldOptions` as the first analysis step — this is the live source of truth for valid field IDs and operators. Never hardcode `custrecord_ngob_*` field IDs in proposed rules.
