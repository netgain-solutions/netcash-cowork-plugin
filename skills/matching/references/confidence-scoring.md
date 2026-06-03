# Confidence Scoring

Score every potential match on a 0-100 scale across four dimensions: Amount, Date, Reference, Entity. Sum the four dimension scores to get the overall match confidence.

## Tiers

| Tier | Range | Action |
|---|---|---|
| **High** | 80-100 | Strong match, minimal risk. Submit after user approval. |
| **Medium** | 60-79 | Good match, some variance. Surface variance explanation prominently. |
| **Low** | 40-59 | Possible match, requires careful review. Present with explicit risk factors. |
| **Below 40** | 0-39 | **Do not suggest.** Confidence is too low to merit user time. |

The 40-point floor is a hard constraint. If your scoring puts a candidate below 40, drop it — don't include it in the proposal even with caveats. Better to leave a transaction unmatched than to invite the user to review noise.

## Per-dimension scoring

### Amount Match (40 points max)

| Condition | Points |
|---|---|
| Exact match | 40 |
| Within $0.01 (rounding noise) | 38 |
| Within $1.00 (penny / fee rounding) | 32 |
| Within 1% | 28 |
| Within 3% (processor fees — Stripe, PayPal, Square) | 22 |
| Within 5% (broader processor fees, partial discounts) | 15 |

Amount carries the most weight because it's the strongest single signal. A perfect amount match anchors a high-confidence proposal; a 3%-off amount match needs a fee explanation in the variance writeup.

### Date Match (20 points max)

| Condition | Points |
|---|---|
| Same date | 20 |
| 1 day difference | 17 |
| 2-3 days | 14 |
| 4-5 days | 10 |
| 6-14 days | 5 |

Date variance is normal — bank settlement lag, weekends, processor batch timing. A 2-3 day gap is the rule rather than the exception. Don't penalize it heavily.

### Reference Match (25 points max)

| Condition | Points |
|---|---|
| Check number exact match | 25 |
| Check number contained in reference | 20 |
| Invoice / PO / Bill number exact match | 22 |
| Partial reference match (e.g. last 4 digits of invoice) | 12 |

Reference match is the cheapest path to high confidence. If a check number on the bank transaction matches the GL transaction's `otherrefnum` or `tranId`, you have a near-certain match even if other fields are noisy.

### Entity Match (15 points max)

| Condition | Points |
|---|---|
| Exact entity match (via NetCash entity mapping) | 15 |
| Entity name contained in merchant string | 12 |
| Partial / fuzzy entity match | 8 |

Entity match alone is rarely decisive (15 points won't clear the 40-floor) but it confirms the other dimensions. A high-amount + high-reference match with no entity fit is suspicious; verify before proposing.

## Worked examples

### Example 1 — High confidence (95)

Bank: Check #4521 for $2,340.00 on 2025-03-15
GL: Bill Payment #4521 to Delta Supplies for $2,340.00 on 2025-03-15

- Amount: 40 (exact)
- Date: 20 (same day)
- Reference: 25 (check number exact match)
- Entity: 10 (Delta Supplies maps to vendor, indirect via mapping)
- **Total: 95 → High**

### Example 2 — Medium with processor variance (73)

Bank: Stripe deposit $970.50 on 2025-03-16
GL/Open AR: Invoice INV-1042 for Acme Corp, $1,000.00, dated 2025-03-14

- Amount: 22 (within 3%, Stripe fee territory)
- Date: 14 (2 days lag, normal for Stripe payouts)
- Reference: 22 (description "STRIPE PAYMENT INV-1042" — invoice number matches)
- Entity: 15 (Acme Corp via mapping, single open invoice for this customer)
- **Variance explanation:** $29.50 difference (~2.95%) matches standard Stripe fee (2.9% + $0.30 = $29.30 expected on $1,000)
- **Total: 73 → Medium**

Processor-fee matches frequently land in the **Medium** tier, not High — there's real variance, and the per-dimension scoring reflects that. Present this kind of match as "high-Medium with strong fee explanation" rather than overstating confidence.

### Example 3 — Medium with strong reference (75)

Bank: ACH withdrawal $7,200.00 on 2025-03-22, description "ACH PMT OMEGA SERVICES"
Open Bill: BILL-2024-092 for Omega Services, $7,200.00, dated 2025-03-18

- Amount: 40 (exact)
- Date: 14 (4 days)
- Reference: 0 (no check number, no bill number in description)
- Entity: 12 (vendor name "Omega Services" contained in description string)
- **Variance explanation:** None — exact amount, ACH settlement lag explains date gap
- **Total: 66 → Medium**

### Example 4 — Low with ambiguity (45)

Bank: Deposit $3,000.00 on 2025-03-20, description "ONLINE DEPOSIT"
Open invoices in same date range:
- INV-2030 for Alpha Inc, $3,000.00
- INV-2031 for Beta LLC, $3,000.00

- Amount: 40 (exact for either invoice)
- Date: 14 (within 2-3 days for either)
- Reference: 0 (description gives no signal)
- Entity: 0 (no entity identifier)
- **Total per candidate: 54 → Low**

**Action:** present both candidates and ask the user to choose. Do not auto-suggest one over the other.

### Example 5 — Below threshold, do not suggest (35)

Bank: Wire $24,000.00 on 2025-03-10, description "INTL WIRE"
Open Invoice: INV-3000 for $25,000.00 on 2025-03-08

- Amount: 15 (within 5%, but not within 3%)
- Date: 14 (2 days)
- Reference: 0 (no reference)
- Entity: 0 (no entity match)
- **Total: 29 → Below 40, drop**

The 4% gap with no fee signal in the description and no entity confirmation is too risky. If the description had said "WIRE FEE $1,000 DEDUCT", the variance explanation would lift it; without that, there's no way to defend the proposal.

## How to apply

When scoring a candidate:

1. Compute each dimension independently using the tables above.
2. Sum to get the total.
3. Apply the tier label.
4. If below 40, drop. If above 40, write the variance explanation.
5. The variance explanation is what justifies a Medium or Low match — without it, drop down a tier or drop entirely.

Confidence scoring is a sanity check on the proposal, not the proposal itself. The user reads your reasoning, not your score. Use the score to decide whether to surface the match; use the reasoning to make it actionable.
