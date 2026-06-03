---
name: investigate-variance
description: Drill into a specific NetCash transaction or variance — pull the data, explain what's there, suggest follow-on if anomaly found.
argument-hint: "[item-or-amount]"
---
You are responding to the user invoking /investigate-variance. The user has identified a specific transaction or variance worth a closer look. Use the `ad-hoc-reporting` skill primarily.

Forward $ARGUMENTS as the investigation scope (could be a transaction ID, an amount, a vendor name, an account, or a free-text description). Pull the relevant data using NetCash tools (`getNetCashBankActivity`, `getNetCashGLActivity`, `findNetCashEntityMappingsByAlias` as appropriate), present what you find, and explain what it means.

If the scope is a reconciliation variance, follow the gating sequence in the `ad-hoc-reporting` skill's "Recon variance investigations" section before pulling data on causes. The framework lives in [`skills/recon-dashboards/references/recon-interpretation.md`](../skills/recon-dashboards/references/recon-interpretation.md) — read it before any recon-related variance investigation.

If the investigation surfaces a probable cause that should be diagnosed further (e.g., the variance maps to a known failure mode like PSP variance or FX rounding), **suggest** that the user invoke `/troubleshoot` for the diagnostic workflow — do not dispatch the troubleshooting skill yourself. The orchestration is sequential and user-initiated; you propose the next step, the user decides whether to take it.
