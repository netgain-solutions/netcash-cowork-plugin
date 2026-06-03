# Processor Fees

Payment processor (PSP) fees are the single most common cause of amount variance between bank deposits and open invoices. When a deposit is roughly 3% less than the invoice amount, the processor fee is almost always the explanation. Recognize the pattern, calculate the expected net, and surface the difference as a variance explanation.

## Standard fee structures

| Processor | Fee | Typical merchant string |
|---|---|---|
| **Stripe** | 2.9% + $0.30 per transaction | `STRIPE`, `STRIPE PAYMENT`, `STRIPE DEPOSIT`, `STRIPE-INV-…` |
| **Stripe Connect** | 2.9% + $0.30 + platform fee (variable) | `STRIPE CONNECT`, `STRIPE PAYOUT` |
| **PayPal** | ~2.9% + $0.30 (varies by country / volume) | `PAYPAL`, `PAYPAL TRANSFER`, `PAYPAL PMT` |
| **Square** | 2.6% + $0.10 per transaction (in-person card) | `SQUARE`, `SQ *MERCHANT` |
| **Square (manually keyed / online)** | 3.5% + $0.15 | same as above |
| **ACH (originator-paid)** | $0.20 – $1.50 per transaction (flat) | varies — `ACH CREDIT`, `ACH DEPOSIT` |
| **ACH (receiver-paid same-day)** | up to $5 flat | `ACH SAMEDAY` |
| **Wire (domestic)** | $15 – $35 flat | `WIRE`, `INCOMING WIRE`, `WIRE FEE` |
| **Wire (international)** | $35 – $50 flat + intermediary deductions | `INTL WIRE`, `IBAN WIRE` |

These are typical retail rates as of 2026. High-volume merchants negotiate lower rates; the customer can confirm their actual rate in their PSP dashboard.

## Calculation

For percentage-based PSP fees:

```
expected_net = invoice_amount × (1 - rate_pct) - flat_fee
fee = invoice_amount - expected_net
```

**Stripe example** ($1,000 invoice):

```
expected_net = 1000 × (1 - 0.029) - 0.30 = 971 - 0.30 = 970.70
fee = 29.30
```

Bank deposit of $970.50 against a $1,000 invoice → fee of $29.50, within $0.20 of the expected $29.30. Strong match.

**PayPal example** ($500 invoice):

```
expected_net = 500 × (1 - 0.029) - 0.30 = 485.50 - 0.30 = 485.20
fee = 14.80
```

**Square in-person example** ($200 invoice):

```
expected_net = 200 × (1 - 0.026) - 0.10 = 194.80 - 0.10 = 194.70
fee = 5.30
```

For flat-fee processors (ACH, wire), the fee is not proportional. Look for fee language explicitly stated in the description (e.g., "$25 FEE DEDUCT").

## Detection rules

When a deposit is between **2.5% and 3.5% less** than an open invoice and the merchant string matches Stripe / PayPal / Square / etc., the processor fee is the most likely explanation. Calculate the expected net and surface the variance explanation.

When a wire transfer is **$10–$50 less** than an open invoice and the description contains fee language ("FEE DEDUCT", "LESS CHARGES", "WIRE CHG"), use the stated fee.

When neither pattern applies but variance is within 5%, do not assume processor fee — surface the variance and ask the user to confirm.

## Match-with-variance template

When a match is explained by a processor fee, include the explicit calculation in the reasoning and use the difference-journal payload to post the fee to the customer's processing-fee GL account.

```
**Match with Variance — Processor Fee**

Bank: $970.50 deposit, Mar 16, "STRIPE PAYMENT INV-1042"
Invoice: INV-1042 for Acme Corp, $1,000.00, Mar 14
Variance: $29.50 (2.95%) — matches Stripe standard fee (2.9% + $0.30 = $29.30 expected on $1,000)

Confidence: 73 (Medium-high)
- Amount within 3%: 22
- Date 2-day lag: 14
- Reference (invoice number in description): 22
- Entity (single open invoice for Acme): 15

Risk: Stripe occasionally bundles multiple charges into one payout. If the deposit feels off, verify by reconciling the Stripe payout report against open invoices for the same period.

Submit payload (post fee to processing-fee account):
{
  "bankSelectedKeys": [123],
  "tranSelectedKeys": [456],
  "isCashApplication": true,
  "manualAction": "bank_cash",
  "isCreateDifferenceJournal": true,
  "offsetAccount": <processing-fee-account-id>,
  "reason": "Stripe deposit $970.50 applies to Invoice INV-1042 for $1,000. $29.50 difference is Stripe processing fee (2.9% + $0.30)."
}
```

## Edge cases

- **Stripe Connect / marketplace splits** — when the merchant is on Stripe Connect, the deposit may include a platform fee that's separate from the standard Stripe fee. Variance can be 4-7% rather than 3%. Look for "STRIPE CONNECT" or "STRIPE PAYOUT" in the description and adjust expectations.
- **Bundled deposits** — Stripe / PayPal / Square deposits often bundle multiple invoices into one bank credit. If the deposit doesn't match a single invoice, sum candidate invoices for the same customer in the same payout period (typically 2-day rolling window) and check whether the sum (less fees) matches.
- **Refunds and reversals** — a Stripe payout that includes a refund will be net of both the original charge and the reversal, with the original fee retained by Stripe. The variance pattern looks like 3% off the gross-of-refund amount, not the net.
- **Cross-currency PSP** — Stripe / PayPal apply FX markup (typically 1-2%) on top of the percentage fee for foreign-currency settlements. For multi-currency customers, expected variance is closer to 4-5%, and the reasoning should call out both the standard fee and the FX markup.
- **High-risk vertical surcharges** — some industries (gaming, adult, certain SaaS) pay elevated processor rates (3.5-4.5%). If the customer is in one of these verticals, adjust expected variance.

## When variance doesn't fit a processor pattern

If the deposit is more than 5% off and no fee language is in the description, do not assume processor fee. Possibilities to consider:

- The deposit is for a different invoice than expected
- The customer paid an old balance plus the current invoice (split across two open items)
- A credit memo or write-off is reducing the expected amount
- FX rounding on a multi-currency match (see `../../troubleshooting/references/multi-currency.md`)

In those cases, surface the unexplained variance and ask the user to clarify rather than asserting a fee match.
