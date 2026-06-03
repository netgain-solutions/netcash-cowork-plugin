---
name: pattern-insights
description: Surface trends, anomalies, and recurring patterns in bank activity that the user might not have noticed — recurring spend, payment processor (Stripe/PayPal/Square) reconciliation patterns, vendor concentration, month-over-month deltas, and unusual activity. Use this skill when the user asks what patterns or trends they should know about, anything unusual on an account, or what's recurring. Cross-references existing automation rules to avoid surfacing patterns already covered.
argument-hint: "[account-name]"
---

## Purpose

Surface what controllers miss in line-by-line views. A controller scrolling through a 30-day register sees individual transactions; this skill steps back and looks at the shape of activity over a window — what repeats, what concentrates, what just spiked, what fees are predictable, what's drifting month-over-month.

The skill is analytical, not actionable. It produces ranked observations and asks "should we automate this?" — it does not create rules, submit matches, or modify any record. When an observation looks automatable, the skill hands off to the `automation-rules` skill (the user invokes `/suggest-rules` or asks directly).

Cross-reference existing automation rules before surfacing anything. A pattern already covered by an Active rule is not an insight — it's the rule working. Filtering those out keeps signal high.

Inherits the plugin-level CLAUDE.md (anti-fabrication, tool routing, OneWorld detection, tone). Skill-specific behavior below.

## Workflow

### Step 1 — Pull the activity data

Call `getNetCashBankActivityByGrouping` to get the aggregated view. Choose the grouping based on the user's question:

- "what's recurring on this account?" / "any patterns?" → group by `merchant` + `month` (or `merchant` + week if lookback < 30 days)
- "where's our spend going?" / "vendor concentration" → group by `merchant` over the full lookback
- "anything weird this month?" / "month-over-month" → group by `category` + `subsidiary` over two periods
- "PSP fees" → group by `merchant` filtered to known PSPs (Stripe, PayPal, Square — see `references/psp-fee-patterns.md` for payee strings)

Default lookback: 90 days if unspecified. Ask the user to confirm the account and lookback before pulling unless they specified both. If the user said "this account" and the conversation already established one, use that.

Use `getNetCashBankActivitySummary` as a sanity-check companion — totals from the summary should reconcile with the sum across groupings. If they don't, surface the discrepancy before proceeding.

### Step 2 — Cross-reference automation rules

Call `getNetCashAutomationRules` for the same account(s). Build a set of merchant + amount + rule-type combinations that are already automated by **Active** rules.

Filter the grouping output: any pattern whose merchant or category is already covered by an Active rule is dropped from the insights surface. A pattern covered only by a Testing-status rule (`status: 2`) is **not** filtered out — surface it with a note that a Testing rule exists, since the user may want to promote it.

If a rule exists but is Inactive (`status: 3` or similar), do not filter — the user disabled it for a reason, and the underlying pattern may now be worth re-evaluating.

### Step 3 — Surface five named insight categories

Run each pattern through the five categories (see Insight Categories below). Each pattern can match more than one category (e.g., a Stripe deposit pattern is both "PSP fee patterns" and may also be "recurring spend"). Tag each observation with all categories that apply; rank by impact, not by category.

### Step 4 — Rank by impact_score

For each observation, compute:

```
impact_score = abs($) over lookback × pattern_confidence
```

`abs($)` is the total absolute dollar magnitude across all instances of the pattern over the lookback window. Use absolute value so large recurring outflows and large recurring inflows are both visible — a controller cares about both.

`pattern_confidence` is a multiplier based on instance count:

- 3 instances → 1.0 (the floor for being called a "pattern" at all)
- 4-7 instances → 1.2
- 8+ instances → 1.5

Tie-break by recency: when impact_scores are within 5% of each other, surface the pattern with the most recent occurrence first. The discontinuity at 3 vs 4 and the cutoffs are explained in `references/anomaly-thresholds.md`.

### Step 5 — Present top 5 with the automation prompt

Present the top 5 observations as a ranked list. For each:

- **Pattern**: 1-line plain-English description ("$12,400 to AWS every month for 6 months")
- **Category**: which of the five named categories it falls into (multiple if applicable)
- **Impact score**: the computed value, with the inputs ($ × multiplier) shown so the user can audit
- **Evidence**: dates + amounts of each instance (link out to the bank activity rows when possible)
- **Already-rule status**: "no rule covers this" / "Testing-status rule #N exists" / "Inactive rule #N exists"

End with: "Should we set up automation rules for any of these? I'll hand off to the `automation-rules` skill if you pick one." Do not invoke `/suggest-rules` or call any rule-creation tool from this skill — the user drives the handoff.

If the user asks about a specific observation ("explain #3", "why is this an outlier?"), drill in: walk through the data, show the comparison set, name the threshold that triggered it. Anti-fabrication rules apply: every dollar amount, date, and entity must come from a tool call, not invented to fit the pattern.

## Insight Categories

Each category has an explicit detection threshold. A pattern that doesn't meet the threshold is not surfaced.

### 1. Recurring spend

**Detection**: same merchant + amount within ±5% of the median for that merchant + cadence within ±5 days of the expected interval, observed ≥ 3 times.

Cadences to test, in order: monthly (28-31 days), quarterly (88-94 days), annual (358-372 days). The first cadence that fits within ±5 days for ≥ 3 consecutive instances wins.

The ±5% amount tolerance handles legitimate variation (price increases, FX drift, partial-period prorating). A merchant whose amount swings > 5% across instances is **not** a recurring-spend pattern — surface it instead under "Month-over-month deltas" or "Unusual activity" depending on shape.

Worked examples and edge cases (Feb 28 vs Mar 31, leap years) in `references/recurring-spend-detection.md`.

### 2. PSP fee patterns

**Detection**: deposits from Stripe, PayPal, or Square (matched by payee string from `references/psp-fee-patterns.md`) where the differential between gross and deposit amount is consistent with the published rate (~2.9% + $0.30 for Stripe/PayPal, 2.6% + $0.10 for Square), observed ≥ 3 times for the same processor.

The differential check is what distinguishes a "pattern" from "we use Stripe and there are deposits". A controller already knows they use Stripe; the insight is "Stripe nets at the published rate every time" or — more interesting — "Stripe's net is varying more than the rate would explain, worth investigating".

Stripe Connect accounts behave differently (the `application_fee` shows as a separate line). Detection rules for Connect, plus per-processor payee-string lists, in `references/psp-fee-patterns.md`.

### 3. Vendor concentration

**Detection**: a single entity accounts for > 20% of total spend over the lookback window.

"Spend" is the sum of absolute outflows (debits) only — concentration on inflows is a different question (revenue concentration, surface differently if asked). Calculate the threshold against the total of all debit transactions in the lookback, then sort entities by their share.

When concentration triggers, also surface the top 3 entities by share even if only the top one crosses 20%. Controllers usually want the context.

The 20% cutoff is documented in `references/anomaly-thresholds.md` along with a note on why 20% (not 10% or 50%): below 20% is too noisy across most controller portfolios; above 50% misses meaningful concentrations like "AWS is 30% of spend".

### 4. Month-over-month deltas

**Detection**: month-over-month change > 30% in **absolute dollar terms** for any merchant or category present in both months. Both filters matter — percentage alone surfaces noise on small line items; absolute alone misses big-percentage shifts on mid-size line items.

"Absolute dollar terms" means the difference between the two months in dollars must exceed some threshold relative to the merchant's typical activity. Concretely: if merchant X averages $5,000/month and this month shows $7,500 (+50%), the delta is $2,500 — that's > 30% **and** materially different from baseline, so it surfaces. If merchant Y averages $50/month and this month shows $200 (+300%), the dollar delta is $150 — surface only if the user explicitly asked about percentages, otherwise filter as noise.

Comparison window: the most recent complete month vs. the prior complete month. If the user is mid-month, compare the partial current month vs. the same partial period in the prior month (e.g., May 1-15 vs April 1-15) and label clearly.

### 5. Unusual activity

**Detection**: a single transaction whose amount exceeds 3× the trailing 90-day standard deviation of the same merchant or category.

This is a per-merchant or per-category z-score-style check, not a portfolio-wide "big number" alert. A $50,000 wire to a vendor that averages $48,000 with σ=$2,000 is not unusual (z = 1.0). A $5,000 wire to a vendor that averages $200 with σ=$50 is unusual (z = 96, well above 3).

If the merchant or category has fewer than 5 prior transactions in the trailing 90 days, the standard deviation isn't reliable — do **not** flag as unusual. Surface instead as "first observed" or "rare merchant" without claiming it's anomalous.

**Don't go silent on sparse data.** If a user asks "anything unusual?" and the lookback doesn't have enough history to support the analysis, say so explicitly: "Not enough history on this account/merchant for an anomaly check (need 5+ prior transactions in the last 90 days; you have N)." The user should know whether silence means "no anomalies" or "couldn't run the check" — those are very different signals.

The 3σ cutoff is documented in `references/anomaly-thresholds.md`.

## Hard Constraints

- **NEVER claim a pattern from < 3 observations.** Two of anything is a coincidence. The pattern_confidence floor is 3 instances; below that, do not call it a pattern, do not compute an impact score, do not surface it. If the user asks about something that only happened twice, describe what you see and explicitly say "this isn't a pattern yet — I'd want to see it once more before treating it as recurring."

- **NEVER recommend a pattern already covered by an Active automation rule.** Step 2 of the workflow filters these out. If the user asks about a specific transaction and you find the rule that handled it, surface the rule, do not pretend the pattern is new. Patterns covered by Testing-status rules are surfaced (with a note); patterns covered by Active rules are filtered.

- **NEVER label as anomaly without explicit comparison data.** "Unusual activity" requires the trailing 90-day baseline. If the merchant is too new to have a baseline (< 5 prior transactions), it's not an anomaly — say "first observed" or "rare merchant" and stop. Do not invent a comparison set.

- **NEVER show an impact_score without explaining how it was calculated when the user asks.** The formula is `abs($) × pattern_confidence` with multiplier tiers (3+ = 1.0, 4-7 = 1.2, 8+ = 1.5). If the user asks "why is this ranked higher?" or "where does that score come from?", show both inputs and the multiplier tier. Do not hand-wave.

## Notes

This skill is invoked either by the explicit `/insights [account]` slash command or by auto-triggering when the user asks about patterns, trends, anomalies, or anything unusual. When other skills (`matching`, `ad-hoc-reporting`, `troubleshooting`) surface a recurring issue, they should suggest the follow-on analysis as a `/insights` invocation rather than dispatching this skill directly.

When this skill surfaces a high-impact observation that looks automatable, suggest the user run `/suggest-rules` for that account — do not dispatch the `automation-rules` skill from inside this one.

References:
- `references/psp-fee-patterns.md` — Stripe / PayPal / Square / Stripe-Connect: net-deposit math, payee strings, detection-from-bank-description rules
- `references/recurring-spend-detection.md` — full thresholds with worked examples for monthly / quarterly / annual cadences and edge-case handling (Feb 28 vs Mar 31, leap years)
- `references/anomaly-thresholds.md` — impact_score formula with the 3-vs-4 discontinuity justification, vendor-concentration cutoff (20%), MoM delta logic (30% absolute dollar change), unusual-activity threshold (3× trailing 90-day σ)
