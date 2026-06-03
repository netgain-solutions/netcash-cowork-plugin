---
name: find-matches
description: Investigate unmatched NetCash bank transactions and find matches — 1:1 GL matches, cash applications to open invoices/bills, and grouped matches.
argument-hint: "[account-name]"
---
You are responding to the user invoking /find-matches. The user wants help finding matches for unmatched bank transactions. Use the `matching` skill. Forward `$ARGUMENTS` as the account scope (or "all accounts" if no argument).

Run the six-pass matching workflow per the skill body. The skill's Mode Selection section determines pacing — read it before starting. Quick summary:

- **Pass-by-pass mode** (default when >15 unmatched bank transactions): pause after each pass that finds matches, present findings, ask permission before submitting and before continuing.
- **Batch mode** (default when ≤15 unmatched, or opt-in at any size): run all six passes back-to-back using read tools only; present one consolidated summary at the end; ask once for a wholesale or partial submit confirmation.

Writes never happen without explicit user confirmation in either mode. Batch mode still pauses for high-risk findings (>$50K, low-confidence near the 40 cutoff, ambiguous many-to-one, cross-currency).

**Argument shape:** `$ARGUMENTS` may include an account name plus an optional mode flag — `--batch` / `--all` / `--continuous` force batch mode; `--step` / `--one-at-a-time` force pass-by-pass. The user can also signal mode in their prose ("run through everything", "go pass by pass", etc.) — the skill recognizes both.

If `$ARGUMENTS` is empty (no account named), ask the user which account they want to investigate before pulling data.
