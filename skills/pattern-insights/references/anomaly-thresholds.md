# Anomaly thresholds

Reference for the impact-ranking formula and the per-category thresholds used by the `pattern-insights` skill. All thresholds here are binding — the skill must apply them as written.

## impact_score formula

```
impact_score = abs($) over lookback × pattern_confidence
```

### `abs($)`

Total absolute dollar magnitude of the pattern across all observed instances within the lookback window. Sum the absolute values of every transaction that's part of the pattern. Use absolute (not signed) values because controllers care equally about large recurring outflows (subscription bloat) and large recurring inflows (PSP deposit patterns, repeat customer payments).

If the pattern is "month-over-month delta" the relevant magnitude is the **delta**, not the gross — use `abs(month_n - month_n-1)`.

### `pattern_confidence` multiplier tiers

| Instance count | Multiplier |
|---|---|
| < 3 | not surfaced (floor — see hard constraints in SKILL.md) |
| 3 | 1.0 |
| 4-7 | 1.2 |
| 8+ | 1.5 |

### Why the discontinuity at 3 vs 4

3 instances is the floor for being called a pattern at all — two is a coincidence, one is a transaction. At 3, we can label it but our confidence in repeatability is low; the 1.0 multiplier reflects no boost beyond the dollar magnitude itself.

At 4+ instances, the pattern has survived an additional cycle of opportunity to deviate. Repeatability confidence is meaningfully higher, justifying the 1.2 boost. At 8+, the pattern has been observed long enough that it's almost certainly a stable structural feature of the account, justifying the 1.5 boost.

The discontinuity (1.0 → 1.2 at instance 4, then 1.2 → 1.5 at instance 8) is intentional. Continuous functions (e.g., `1 + 0.05 × log(n)`) would be more elegant but harder for users to audit. Stepwise multipliers let the user verify the score by hand: see 5 instances → multiplier is 1.2 → done. The mental-math accessibility wins.

### Tie-break

When two impact_scores are within 5% of each other, the more recent pattern wins (latest occurrence first). Recency is signal: a pattern with its last occurrence yesterday is more decision-relevant than a pattern whose last occurrence was 80 days ago, even at equal magnitude.

## Per-category thresholds (binding)

### Recurring spend

Same merchant + amount within ±5% of merchant's median + cadence within ±5 days of expected interval, observed ≥ 3 times. Full thresholds and worked examples in `recurring-spend-detection.md`.

### PSP fee patterns

Stripe / PayPal / Square deposits with consistent fee differential (within ±0.5pp of published or customer-specific observed rate), observed ≥ 3 times for the same processor. Full per-processor rates and payee strings in `psp-fee-patterns.md`.

### Vendor concentration

A single entity accounts for **> 20% of total spend** over the lookback window.

"Spend" = sum of absolute outflows (debit transactions) only. Calculate against total debit volume; sort entities by share. The top entity surfaces if and only if it crosses 20%; when it does, also surface the next 2 entities by share regardless of whether they cross the threshold (controllers want context).

**Why 20%, not 10% or 50%:**

- **10% would be too noisy.** In a typical controller's portfolio of operating expenses, several vendors will routinely sit between 10-15%. Flagging at 10% generates more false positives than insight.
- **50% would miss meaningful concentrations.** Many real customer environments have a single vendor at 25-35% (often AWS, Stripe net deposits, or a major supplier). Concentration at this level is decision-relevant — refinancing terms, contingency planning, vendor-risk reviews — and a 50% threshold misses it entirely.
- **20% is the empirical break.** Above 20%, concentration is uncommon enough to be noteworthy and material enough to warrant action. This is heuristic, not derived from data — revisit if pilot feedback shows a different break is more useful.

### Month-over-month deltas

Month-over-month change > **30% in absolute dollar terms** (not just percentage) for any merchant or category present in both months.

The "absolute dollar terms" requirement filters out small-base noise. A merchant going from $100 to $200 is +100% but only $100 in absolute terms — not material. A merchant going from $50,000 to $65,000 is +30% and $15,000 in absolute terms — material at most controller scales.

**Algorithm:**

1. Compute `delta = abs(month_n - month_n-1)` for each merchant or category.
2. Compute `pct_change = delta / month_n-1` (or `month_n` if `month_n-1` is zero).
3. Surface if `pct_change > 30%` **AND** `delta` is material at the account scale (heuristic: top 50% of all deltas observed in the comparison, or > $1,000 absolute, whichever is stricter).

The "material at account scale" filter is to avoid surfacing 30 trivial alerts on a high-volume account. Tune in pilot.

**Comparison window:**

- Default: most recent complete calendar month vs. prior complete calendar month.
- Mid-month case: partial current month (e.g., May 1-15) vs. same partial period in the prior month (April 1-15). Label clearly that this is a partial-period comparison.

### Unusual activity

A single transaction whose amount exceeds **3× the trailing 90-day standard deviation** of the same merchant or category.

Per-merchant or per-category z-score, not portfolio-wide. The trailing 90-day window is the baseline; if the merchant or category has fewer than 5 prior transactions in that window, the standard deviation is unreliable and the transaction is **not** flagged as unusual.

**Algorithm:**

1. For the transaction in question, identify its merchant or category.
2. Pull the trailing 90-day history for that merchant or category.
3. If `count < 5`, skip — surface as "first observed" or "rare merchant" with no anomaly claim.
4. Compute `mean` and `std` over the trailing 90-day amounts.
5. Compute `z = abs(transaction_amount - mean) / std`.
6. Flag if `z > 3`.

**Why 3σ, not 2σ or 4σ:**

- 2σ flags ~5% of transactions in a normal distribution → too noisy
- 3σ flags ~0.3% → matches "this is genuinely surprising" intuition
- 4σ flags ~0.006% → too restrictive; misses the soft-anomaly cases controllers want to see

3σ is the standard-deviation convention in most anomaly-detection literature; it's the right tradeoff for surfacing roughly 1-2 anomalies per typical month per account, which matches what controllers find actionable in pilot conversations.

## When to drop a pattern from ranking

Beyond the thresholds, these conditions remove a pattern from the surface entirely:

- Already covered by an Active automation rule (filter at workflow Step 2)
- Refund or reversal pair (a credit + debit of the same amount within 7 days for the same merchant) — the net is zero; surfacing either side alone is misleading
- Inter-account transfers between the customer's own bank accounts — these are handled by `Create Transfer` rules and don't represent vendor or category activity

Patterns covered by Testing-status rules (`status: 2`) are **not** dropped; surface them with a "Testing rule exists, would you like to promote it to Active?" framing.

## Audit trail

When the user asks "why is this ranked higher?" or "show me the math", surface:

1. The pattern's `abs($)` total
2. The instance count
3. The multiplier tier that was applied (3 → 1.0, 4-7 → 1.2, 8+ → 1.5)
4. The product (`abs($) × multiplier = impact_score`)
5. If applicable, which other patterns it's tie-broken against and why (recency)

Never show a score without being prepared to defend it from inputs.
