# Dashboard Templates

Output templates for the four view modes in the `recon-dashboards` skill. Each template includes: what gets surfaced and why, output structure, an example, sort order, and truncation rules.

These are the canonical formats. The skill's Output Format section in `SKILL.md` defers to this file for the specifics so the SKILL body can stay short.

---

## Template 1 — Daily check

**Purpose.** A 2-minute morning scan. Three things the controller wants to know before they pour their second coffee: what's new, what's stale, what's stuck.

**What gets surfaced and why.**

| Section | Source tool | Why it matters |
|---|---|---|
| What changed since last sync | `getNetCashBankActivity` (last 24h by `lastUpdatedDate`) | Catches new ingestion before the user manually opens the recon module; flags sync failures (zero new = sync gap) |
| Outstanding > 7 days | `getNetCashBankActivity` (`isOnlyUnmatched=true`, `transactionDateBefore` = today − 7d) | Items aging past 7 days are the leading indicator of recon backlog; controllers want them surfaced before they become 30-day items |
| Accounts with stale recon | `getNetCashReconciliations` (`in-progress` / `not-started`, `lastUpdatedDate` > 5 business days) | A reconciliation that hasn't moved in 5+ business days is stuck; the controller may have forgotten about it |

**Output structure.**

```
Reconciliation dashboard — daily check view

Headline (bullet list — one item per signal, each item ≤12 words):
- 3 new transactions overnight on Operating Checking
- 8 unmatched items aging > 7 days across all accounts
- 1 recon stuck > 5 business days (Operating Checking, April)
- 1 account with no April recon started (Money Market)

### What changed since last sync
Last sync: <ISO timestamp>. Hours since last sync: <N>. [Staleness warning if > 24h.]

| Account | New | Auto-matched | Unmatched |
|---|---|---|---|
| Operating Checking | 14 | 11 | 3 |
| Payroll Checking | 2 | 2 | 0 |
| Stripe Clearing | 8 | 6 | 2 |

### Outstanding > 7 days (top 10 of <total>)

| Account | Date | Amount | Description | Days outstanding |
|---|---|---|---|---|
| Operating Checking | 2026-04-23 | -$12,450 | ACH WIRE STRIPE PAYOUT | 12 |
| Stripe Clearing | 2026-04-21 | $8,932 | DEPOSIT REF #SP-44192 | 14 |
| ... | | | | |

### Stale reconciliations

| Account | Period | Status | Last activity | Days stuck |
|---|---|---|---|---|
| Operating Checking | 2026-04 | in-progress | 2026-04-26 | 9 |

Suggested next: `/find-matches Operating Checking` to investigate the outstanding ACH WIRE STRIPE PAYOUT.
```

**Sort order.**
- "What changed since last sync" — alphabetical by account name (consistent ordering, no drama).
- "Outstanding > 7 days" — `transactionDate` ascending (oldest first).
- "Stale reconciliations" — `daysStuck` descending (most stuck first).

**Truncation rules.**
- "What changed" — show all accounts in scope; no truncation.
- "Outstanding > 7 days" — top 10. If more, note total count and suggest `/exception-summary` for the full list.
- "Stale reconciliations" — show all stale. There should rarely be more than 5; if there are, the user has bigger problems and they want to see them.

---

## Template 2 — Cash position

**Purpose.** "What does each account hold, and which accounts have recon work outstanding?" Per-account `accountBalance`, `glBalance`, and `variance` from `getNetCashAccountBalances` (live, as of today or a named date — bank **and** credit-card accounts), joined to recon `status` from `getNetCashReconciliations`. The **status column is the "needs attention" signal** — *not* the GL/Account balance gap (`variance`), which is normal whenever outstanding items are in flight. See `references/recon-interpretation.md` for the field semantics and the failure mode this view design avoids.

**What gets surfaced and why.**

| Section | Source tool | Why it matters |
|---|---|---|
| Data source label | Always | Tells the user the as-of date of the balances — leads with "Balances as of <date>" |
| Per-account balances | `getNetCashAccountBalances` (defaults to today; `asOfDate` for a named date) | Returns `accountBalance`, `glBalance`, and `variance` (signed GL − Account) per bank **and** credit-card account — live, mirroring the dashboard balance widget. Use directly; never derive. |
| Recon status | `getNetCashReconciliations(columnSet='detailed')`, joined on account ID | Supplies the `status` "needs attention" signal (and, on OneWorld, the subsidiary each account rolls up to). Use `detailed` — it returns both `status` and `subsidiary`; `financial` omits `status`. Not used for balances. |
| Subsidiary totals (OneWorld only) | Aggregated from per-account balances | Controllers running OneWorld want per-legal-entity totals |

**Output structure (non-OneWorld).**

```
Reconciliation dashboard — cash position
Balances as of 2026-05-31.

Headline (bullet list):
- $3.50M total bank cash across 3 bank accounts
- 3 Brex cardholder cards — $62.6K combined card spend, sharing one GL account ($16.7K GL liability)
- 1 account flagged for attention: Operating Checking (Balance Changed — reopen and re-review)
- 2 bank accounts cleanly Completed

### Bank accounts

| Account | Account Balance | GL Balance | Variance (net, signed GL − Account) | Status |
|---|---|---|---|---|
| Operating Checking (1234) | $250,000.00 | $205,000.00 | -$45,000.00 | Balance Changed |
| Money Market (5678) | $2,500,000.00 | $2,500,000.00 | $0.00 | Completed |
| Reserve Savings (9012) | $750,000.00 | $750,000.00 | $0.00 | Completed |
| **Total bank cash** | **$3,500,000.00** | **$3,455,000.00** | | |

### Credit-card accounts

Cards that share a GL account (common with Brex/Ramp — each cardholder is its own account row, all rolling up to one GL liability account) repeat that one GL balance on every row. Show **Account Balance per cardholder**, then the shared **GL balance and its variance once** on a GL-account subtotal row — never sum the repeated GL across cardholders. A card with its own GL account renders as a normal row, like the bank table above.

| Account | Account Balance | GL Balance | Variance (signed GL − Account) | Status |
|---|---|---|---|---|
| Brex — Cardholder 1 | $17,800.00 | | | — |
| Brex — Cardholder 2 | $18,900.00 | | | — |
| Brex — Cardholder 3 | $25,900.00 | | | — |
| **Company Brex Account (GL 1060)** | **$62,600.00** | **-$16,700.00** | **-$79,300.00** | Completed |

Note on the columns: **GL Balance** comes straight from `getNetCashAccountBalances` — no derivation. **Variance** is `glBalance − accountBalance` (signed GL minus Account) — the net of outstanding GL items (checks in flight, DIT) and outstanding bank items. A non-zero value is **normal** for any account with checks-in-flight or deposits-in-transit; do not present it as a problem. **Status** (joined from `getNetCashReconciliations`) is the actual "needs attention" signal — see `references/recon-interpretation.md` § "Balance Changed status". Bank cash and credit-card balances are totaled **separately** — a credit-card balance is a liability, never add it into bank cash. When cards share one GL account (e.g. several Brex cardholders → one company Brex GL account), each card row repeats that single GL balance — total **Account Balance** across the cardholders but show the **GL balance once per GL account** (a GL-account subtotal row, with its variance = GL − combined card balance); never sum the repeated GL across cardholder rows.

Suggested next: `/investigate-variance Operating Checking` to reopen the Balance Changed recon and identify which transactions changed since submission.
```

**Output structure (OneWorld).** Add a "Subsidiary totals" table above the per-account view, aggregating balances by subsidiary (the subsidiary each account rolls up to comes from the `getNetCashReconciliations` join, or pass `consolidatedSubsidiary` to `getNetCashAccountBalances` for base-currency consolidation). Detect OneWorld per the rule in plugin-level `CLAUDE.md` `<oneworld_detection>` (`ns_getSubsidiaries`, falling back to a `ns_runCustomSuiteQL` subsidiary count), cached per session; on single-subsidiary instances, omit the subsidiary table and surface that you did so. Keep bank and credit-card totals separate within each subsidiary.

**Sort order.**
- Per-account balances — within each section (bank, credit-card), accounts whose status is not `Completed` / `Auto Reconciled` first (sorted by status priority: Balance Changed > Stuck > Not Started > In Progress > Submitted for Review). Remaining accounts next, sorted by Account Balance descending.
- Subsidiary totals (OneWorld) — total bank cash descending.

**Truncation rules.** No truncation. The view is a balance snapshot. If the customer has 50 accounts, show 50.

**What this template does NOT do.**

- **Does not compute balances from `getNetCashBankActivitySummary`.** Forbidden by SKILL.md Hard Constraints — that tool returns cash-flow sums, not balances. Balances come from `getNetCashAccountBalances`.
- **Does not derive GL Balance.** `glBalance` comes straight from `getNetCashAccountBalances`; never compute it as `bank_balance + balance_difference`.
- **Does not flag accounts solely because `variance != 0`.** That's the GL − Account gap, normal whenever outstanding items exist. The **Status** column (joined from `getNetCashReconciliations`) drives "needs attention" flagging. See `references/recon-interpretation.md` for the framework.
- **Does not sum credit-card balances into bank cash.** Credit-card balances are liabilities; total them separately from bank cash.
- **Does not sum a shared GL balance across cardholder rows.** When several cards map to one GL account (Brex/Ramp), `getNetCashAccountBalances` repeats that GL balance on every card row — sum Account Balance per cardholder, but show the GL once per GL account. Summing it across cards multiplies the liability by the cardholder count.

---

## Template 3 — Recon status

**Purpose.** "Where are we in the close?" Per-account view of which reconciliations are done, in-progress, not-started, or reopened.

**What gets surfaced and why.**

| Section | Source tool | Why it matters |
|---|---|---|
| Status summary | `getNetCashReconciliations` (current period, all statuses) | One-glance view of "are we 5 done out of 8 or 1 done out of 8?" |
| In-progress detail | `getNetCashReconciliations` filtered to `in-progress` | Controllers want to know what's stuck and why |
| Match progress | `getNetCashReportMatches` enrichment (works on all instances; subsidiary breakdown only renders on OneWorld) | "Recon X is in-progress and 80% matched" is more useful than "Recon X is in-progress" |

**Output structure (OneWorld).**

```
Headline: "5 of 8 accounts complete for April 2026. 2 in-progress (1 stuck >5 days), 1 not started."

### Status summary — Period 2026-04

| Status | Count | Accounts |
|---|---|---|
| complete | 5 | Payroll Checking, Reserve Savings, Stripe Clearing, EU Operating Checking, EU Reserve Savings |
| in-progress | 2 | Operating Checking, APAC Operating |
| not-started | 1 | APAC Reserve |
| reopened | 0 | — |

### In-progress detail

| Account | Subsidiary | Started | Days in state | Match progress | Outstanding $ |
|---|---|---|---|---|---|
| Operating Checking | US Operations | 2026-04-26 | 9 (stuck) | 142/167 (85%) | -$84,200 |
| APAC Operating | APAC Operations | 2026-05-02 | 3 | 67/89 (75%) | $12,300 |

### Not started

| Account | Subsidiary | Period |
|---|---|---|
| APAC Reserve | APAC Operations | 2026-04 |

Suggested next: `/find-matches Operating Checking` to investigate the 25 unmatched transactions on the stuck recon.
```

**Output structure (non-OneWorld).** Keep the "Match progress" column from the in-progress detail (`getNetCashReportMatches` still works). Drop the Subsidiary column. Note at the bottom of the view that the subsidiary breakdown isn't applicable on a single-subsidiary instance.

**Sort order.**
- Status summary — fixed order: `not-started`, `in-progress`, `reopened`, `complete`.
- In-progress detail — days-in-state descending (most stuck first). Tie-break by outstanding $ descending.
- Not-started — alphabetical by account.

**Truncation rules.** No truncation for current-period status. If the user expands to multiple periods, default to last 3 closed periods plus current.

---

## Template 4 — Exception summary

**Purpose.** "What's the biggest break, and the next biggest, and the next?" Top-N exceptions ranked by impact.

**What gets surfaced and why.**

| Section | Source tool | Why it matters |
|---|---|---|
| Top exceptions | `getNetCashBankActivity(isOnlyUnmatched=true)` | The single most-asked controller question is "what are my biggest unmatched items?" — this answers it directly |
| Break-type classification | Inferred from description + amount pattern | A list of dollar amounts is less useful than a list of dollar amounts with "PSP fee variance" or "transfer pending" labels |
| Aggregate counts | Computed from the same source | Context: "20 exceptions over $1K, 5 over $10K, 1 over $100K" |

**Break-type classification table.**

The classifier is heuristic. If a transaction matches multiple, pick the most specific. If it matches none, label "needs investigation" — do not invent a break type.

| Pattern | Description signal | Amount signal | Break type label |
|---|---|---|---|
| PSP payout from Stripe / PayPal / Square | Description contains `STRIPE`, `PAYPAL`, `SQUARE` | Net is ~97-99% of expected gross | PSP fee variance |
| Wire / ACH transfer between own accounts | Description contains `TRANSFER`, `INTERNAL`, or two of the org's own bank names | Same dollar amount on both sides within 1-2 business days | Transfer pending |
| Currency mismatch | Account currency differs from transaction currency | Amount converts to within 0.5% of a known GL transaction | FX timing / FX rounding |
| Round-number deposit, no description detail | Description is `DEPOSIT` / `CREDIT` only, no reference | Round number ($1000, $5000, $10000) | Manual deposit, needs entity |
| Round-number withdrawal | Description is `CHECK PAID` or `WITHDRAWAL`, no reference | Round number | Check pending GL entry |
| Recurring vendor outflow | Description contains a known recurring vendor | Same as last month ±5% | Recurring spend, no rule |
| Anything else | — | — | Needs investigation |

**Output structure.**

```
Headline: "20 unmatched exceptions across 3 accounts, $284,500 total. Top 3 are PSP fee variance ($45K), transfer pending ($38K), check pending GL entry ($28K)."

### Aggregate

| Tier | Count | Total $ |
|---|---|---|
| > $100K | 1 | $142,000 |
| $10K - $100K | 5 | $98,300 |
| $1K - $10K | 14 | $44,200 |
| Total | 20 | $284,500 |

### Top 20 exceptions

| Rank | Account | Date | Amount | Description | Break type |
|---|---|---|---|---|---|
| 1 | Operating Checking | 2026-04-15 | -$142,000 | WIRE OUT REF#WX-99231 | Needs investigation |
| 2 | Stripe Clearing | 2026-04-23 | $48,300 | STRIPE PAYOUT MAY1 | PSP fee variance |
| 3 | Operating Checking | 2026-04-20 | -$38,000 | TRANSFER TO RESERVE SAVINGS | Transfer pending |
| 4 | Payroll Checking | 2026-04-18 | -$28,400 | CHECK 4422 PAID | Check pending GL entry |
| 5 | EU Operating Checking | 2026-04-22 | EUR -22,000 | WIRE OUT VENDOR INVOICE | FX timing |
| ... (rows 6-20) | | | | | |

Suggested next: `/troubleshoot stripe payout variance` for the top PSP fee variance, or `/find-matches Operating Checking` for the unexplained $142K wire.
```

**Sort order.**
- Aggregate tiers — fixed order: descending dollar tier.
- Top 20 — absolute amount descending. Tie-break by transaction date ascending (older first within the same amount).

**Truncation rules.**
- Show top 20. If more, note total count: "20 of 47 exceptions shown".
- If aggregate has zero items in a tier, omit that tier row (don't show "$10K-$100K: 0").

---

## Multi-account scoping note

If `$ARGUMENTS` provides a single account, all four views scope to that account and the templates collapse where applicable (e.g., daily check has one row in "What changed", not multiple).

If no account is provided, scope to all active accounts from `getNetCashBankAccounts`. If the user has more than 12 active accounts, surface a note suggesting they run with a specific `account-name` argument for a faster, more focused view.