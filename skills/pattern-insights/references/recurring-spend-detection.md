# Recurring spend detection

Reference for detecting recurring spend (subscriptions, retainers, recurring vendor payments) in NetCash bank activity. Used by the `pattern-insights` skill as one of the five named insight categories.

## Detection thresholds (binding)

A pattern qualifies as "recurring spend" when **all three** conditions hold:

1. **Same merchant**, identified by `custrecord_ngob_banktran_merchant` (preferred) or by alias resolution via `findNetCashEntityMappingsByAlias` when merchant is empty
2. **Amount within ±5%** of the median amount for that merchant across observed instances
3. **Cadence within ±5 days** of the expected interval, observed ≥ 3 consecutive times

Drop any of these conditions and the observation does not surface as a recurring-spend insight. Below 3 instances, the floor in the parent skill applies — do not call it a pattern.

## Cadences to test

In order of priority, test for fit:

| Cadence | Expected interval | Tolerance window |
|---|---|---|
| Monthly | 28-31 days | ±5 days, so 23-36 days |
| Quarterly | 88-94 days | ±5 days, so 83-99 days |
| Semi-annual | 180-185 days | ±5 days, so 175-190 days |
| Annual | 358-372 days | ±5 days, so 353-377 days |

The first cadence that fits ≥ 3 consecutive instances wins. If no cadence fits, the merchant is not classified as recurring even if the same dollar amount appears multiple times — non-cadenced repeat charges (e.g., usage-based monthly variability with sporadic timing) are surfaced under "Vendor concentration" or "Month-over-month deltas" instead.

## Worked examples

### Monthly, clean

Vendor: AWS. Lookback: 6 months.

| Date | Amount | Days since prior | Within ±5%? |
|---|---|---|---|
| 2025-11-15 | $12,400.00 | — | (median = $12,475) |
| 2025-12-15 | $12,520.00 | 30 | yes (+0.4%) |
| 2026-01-15 | $12,475.00 | 31 | yes (0%) |
| 2026-02-14 | $12,395.00 | 30 | yes (-0.6%) |
| 2026-03-14 | $12,610.00 | 28 | yes (+1.1%) |
| 2026-04-15 | $12,580.00 | 32 | yes (+0.8%) |

6 instances, median $12,475, max deviation +1.1%, cadence 28-32 days (within monthly tolerance). Qualifies as recurring spend, monthly. pattern_confidence = 1.2 (4-7 instances). impact_score = $74,980 × 1.2 = $89,976.

### Quarterly, with a price bump

Vendor: Marketing Agency. Lookback: 12 months.

| Date | Amount | Days since prior | Within ±5%? |
|---|---|---|---|
| 2025-05-01 | $25,000.00 | — | (median = $25,000) |
| 2025-08-01 | $25,000.00 | 92 | yes (0%) |
| 2025-11-01 | $25,000.00 | 92 | yes (0%) |
| 2026-02-01 | $26,500.00 | 92 | yes (+6.0%) — **fails ±5% test** |

The 2026-02-01 instance is +6.0% from the prior median — exceeds ±5%. Two interpretations:

1. **Price increase**: the amount is the new normal. Re-evaluate by recomputing the median across all instances ($25,375). The 2026-02-01 amount is now +4.4% from the new median — within tolerance — but the prior three instances are -1.5%, all also within tolerance from the new median. With the recomputed median, the pattern qualifies, but flag the price bump as a sub-finding when surfacing.

2. **Anomaly**: the amount is a one-off (e.g., extra invoice). Re-evaluate by treating this instance as separate. The prior three instances are still a pattern (3 instances, monthly amount, quarterly cadence). Surface this as quarterly recurring spend with a one-off variance flagged for investigation.

Default behavior: assume #1 (price bump) if the new amount holds for the next instance. If the next instance reverts to ~$25,000, retroactively classify as #2 (anomaly). Be explicit about which interpretation was used.

### Annual, with a leap-year edge

Vendor: Annual SaaS Subscription. Lookback: 4 years.

| Date | Amount | Days since prior |
|---|---|---|
| 2023-02-28 | $50,000.00 | — |
| 2024-02-29 | $50,000.00 | 366 (leap year) |
| 2025-02-28 | $50,000.00 | 365 |
| 2026-02-28 | $50,000.00 | 365 |

The 2024-02-29 → 2025-02-28 transition is exactly 364 days (leap day skipped); 2025-02-28 → 2026-02-28 is exactly 365. All within annual tolerance (358-372 ±5). Qualifies as annual recurring.

### Edge case: Feb 28 vs Mar 31

A merchant billed on the last day of every month creates a non-uniform interval:

| Date | Days since prior |
|---|---|
| 2026-01-31 | — |
| 2026-02-28 | 28 |
| 2026-03-31 | 31 |
| 2026-04-30 | 30 |
| 2026-05-31 | 31 |

All intervals are 28-31 days (within monthly tolerance ±5). Qualifies as monthly. Don't be tripped up by the 28-day Feb interval — that's still inside the monthly window.

### Edge case: leap-year shift

If a vendor bills on a strict 30-day cycle (uncommon, but exists for some retainer agreements), February rolls forward differently in leap years vs non-leap years. Detection still passes because ±5 days tolerance absorbs it. Surface a note only if the cadence has visibly drifted by > 7 days over the lookback (e.g., the merchant moved from "first of month" to "fifth of month" gradually).

## Cadence-detection algorithm (for the skill)

Given a merchant's transaction list sorted by date:

1. Compute intervals between consecutive instances.
2. Test each cadence in priority order (monthly first).
3. A cadence "fits" if ≥ 3 consecutive intervals fall within the cadence's tolerance window.
4. Return the first fitting cadence.
5. If no cadence fits but the merchant has ≥ 3 same-amount instances at irregular intervals, classify as "repeat charges, non-cadenced" — do not surface as recurring spend; defer to vendor-concentration or MoM-delta categories.

## What this excludes (intentional)

- Single-merchant patterns that vary > ±5% in amount → not recurring; usually usage-based (cloud, telecom) and belong under "Vendor concentration"
- Same merchant + same amount at irregular intervals → not recurring; defer to MoM deltas if material
- Internal transfers between own accounts → exclude entirely from recurring-spend detection (they're handled by `Create Transfer` rules in the `automation-rules` skill)
- Refunds and reversals → exclude; they distort the median and cadence calculations
