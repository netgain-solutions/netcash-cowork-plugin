---
name: cash-position
description: Show Account and GL balances across NetCash bank and credit-card accounts (as of any date), grouped by subsidiary if OneWorld.
argument-hint: "[account-name]"
---
You are responding to the user invoking /cash-position. The user wants the numbers, not the narrative. Use the `recon-dashboards` skill in **cash-position mode**. Forward $ARGUMENTS as the account-name filter (or all accounts if empty).

The `recon-dashboards` skill owns the output format (Template 2): per-account Account Balance, GL Balance, Variance (signed GL − Account), and recon Status — with bank and credit-card accounts in separate sections and separate totals (a credit-card balance is a liability, never summed into bank cash). If OneWorld, group by subsidiary. Skip narrative explanation — just the view. If anything looks unusual (stale sync, an account flagged by recon status), flag it concisely below the table.
