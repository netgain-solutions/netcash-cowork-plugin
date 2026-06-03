# PSP fee patterns

Reference for detecting payment-service-provider (PSP) deposit + fee patterns in NetCash bank activity. Used by the `pattern-insights` skill to surface PSP fee patterns as a named insight category, and cross-referenced by the `matching` skill for fee-explained variance matches.

## Published fee rates (2026)

| Processor | Standard online rate | In-person rate | Notes |
|---|---|---|---|
| Stripe | 2.9% + $0.30 per successful charge | 2.7% + $0.05 (Stripe Terminal) | International cards add 1.5%, currency conversion adds 1% |
| PayPal | 2.9% + $0.30 per transaction (US domestic) | varies by reader | Cross-border fees stack: typically +1.5% or more |
| Square | 2.6% + $0.10 (in-person, swiped/dipped/tapped) | 3.5% + $0.15 (manually keyed) | Online: 2.9% + $0.30 |
| Stripe Connect | Same base rate; platform's `application_fee` shown separately | n/a | Two-leg structure: gross → connected account; platform fee → platform |

These rates are the published standard. Customer-specific negotiated rates exist for higher-volume merchants — if a customer's actual fee differential is consistently lower than published, treat their negotiated rate as the baseline (after observing it ≥ 3 times) instead of the published rate.

## Net deposit calculation

For a single charge:

```
net_deposit = gross - (gross × pct_rate) - flat_fee
```

For a batch deposit (typical — PSPs settle daily or weekly in batches):

```
net_deposit = sum(gross_i) - sum(gross_i × pct_rate) - (count × flat_fee)
            = sum(gross) × (1 - pct_rate) - count × flat_fee
```

The `count × flat_fee` term is what causes "the fee differential isn't a clean percentage" surprises. A batch of 100 small transactions has $30 of flat-fee component on top of the percentage; the same gross total in 5 large transactions only has $1.50.

## Detecting PSP deposits from bank description

Common payee strings (case-insensitive, substring match):

**Stripe:**
- `STRIPE`
- `STRIPE TRANSFER`
- `STRIPE PAYMENTS`
- `ST-` followed by alphanumeric (Stripe transfer ID prefix on some bank feeds)

**PayPal:**
- `PAYPAL`
- `PAYPAL TRANSFER`
- `PP*` (when followed by a merchant name — PayPal-routed customer payments)
- `PAYPAL DES:TRANSFER` (ACH descriptor format)

**Square:**
- `SQUARE INC`
- `SQ *` (when followed by a merchant name — Square-routed payments)
- `SQUARE TRANSFER`

**Stripe Connect (platform receivables):**
- `STRIPE` + the platform's connected-account merchant name (varies — surface the connected-account name as a follow-up question if Connect is suspected)
- `application_fee` line items show in the Stripe dashboard but **not** in the bank feed; bank only sees the net transfer

If the bank description matches one of the strings above but the deposit amount doesn't reconcile with any expected gross batch, surface the discrepancy under "PSP fee patterns" with a `pattern_confidence` of 1.0 (3 instances) only after seeing the same anomaly 3+ times. Single-instance variance belongs in the `matching` skill, not here.

## Detection threshold for the pattern-insights skill

To label something as a "PSP fee pattern" insight:

1. ≥ 3 deposits from the same processor within the lookback window
2. Each deposit's gross-to-net differential is consistent with the published (or customer-specific observed) rate within ±0.5 percentage points
3. The pattern is not already covered by an Active GL Match rule with a fee tolerance (check `getNetCashAutomationRules` filtered to the bank account)

A consistent-rate pattern should be promoted to an automation rule (`/suggest-rules` for that account). A drifting-rate pattern (differential variance > ±0.5pp across instances) is the more interesting insight — surface it with the variance shown, since it usually means the processor changed terms, the customer added cross-border volume, or there's a category mismatch in NetSuite.

## Worked example — Stripe daily settlement

Bank deposits over 5 business days from `STRIPE TRANSFER`:

| Date | Bank deposit (net) | Stripe gross batch | Differential | Implied fee rate |
|---|---|---|---|---|
| 2026-04-21 | $9,704.20 | $10,000.00 | $295.80 | 2.96% |
| 2026-04-22 | $4,853.10 | $5,000.00 | $146.90 | 2.94% |
| 2026-04-23 | $7,278.65 | $7,500.00 | $221.35 | 2.95% |
| 2026-04-24 | $1,940.90 | $2,000.00 | $59.10 | 2.96% |
| 2026-04-25 | $2,912.55 | $3,000.00 | $87.45 | 2.92% |

5 instances, all within ±0.5pp of the published 2.9% baseline (the +flat-fee component shows as the slight lift to ~2.95%). Pattern confidence: 1.2 (4-7 instances tier). impact_score = $26,690 × 1.2 = $32,028. This pattern is automatable as a GL Match rule with a percentage-based amount tolerance.

## Stripe Connect note

Stripe Connect platforms see two different patterns depending on the platform's `charge_type`:

- **Direct charges**: connected account is the merchant of record. Bank feed shows the net transfer **to the connected account**, not to the platform. The platform sees only its `application_fee` aggregated as a separate transfer.
- **Destination charges**: platform is the merchant of record, transfer to connected account is a separate line. Bank feed shows the gross from customer + the transfer-out to connected account; the difference is the platform's revenue.

If the customer is on Connect, ask which charge type they use before calling the pattern. Cross-charge-type aggregation produces nonsense fee rates.
