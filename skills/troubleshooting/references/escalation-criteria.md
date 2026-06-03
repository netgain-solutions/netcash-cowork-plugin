# Escalation Criteria

When to recommend the user contact Netgain Support, and what information to include in the ticket. The default disposition for any NetCash issue is to walk the named-failure-mode diagnostic in `known-failure-modes.md` — escalation is for the cases where that walk has run out of road.

---

## Escalate when any of the following is true

### 1. Repeated script timeouts

A NetCash UI page, scheduled script, or map/reduce script times out more than once on the same operation. Symptoms include:

- "SSS_TIME_LIMIT_EXCEEDED" errors in the user's view
- A scheduled NetCash automation script (Bank Automation, Reconciliation, etc.) that runs past its window and fails
- A map/reduce post-sync chain that doesn't complete

Don't keep retrying as troubleshooting. Surface to the user that this needs Netgain Support engagement.

### 2. Pages loading > 3-5 minutes

A NetCash UI page (Match Assist, Reconciliation, Bank Activity) that takes more than 3-5 minutes to load is past the threshold where a configuration tweak will fix it. Likely root causes (data-volume, indexing, custom-field overhead) require Netgain Support inspection of the customer's tenant.

### 3. Unexplained data corruption

A match record, bank transaction, or GL transaction in an inconsistent state that is not explained by any of the five named patterns (`known-failure-modes.md`) and is not explained by a recent user action the user remembers. Surface to support; don't recommend a manual fix that might compound the corruption.

Examples that qualify:
- A match record exists in the recon UI but the linked transactions are inactive or deleted
- A bank transaction shows as matched but `getNetCashReportMatches` returns no record for it
- Recon balance ties out by ledger but the recon page shows a different number

### 4. Bank feed failures after troubleshooting

The user has confirmed via the bank's portal that transactions exist for a date, but those transactions are not in NetCash. Troubleshooting steps already attempted:

- Re-import for the affected date, or re-trigger sync
- Verified the bank account is correctly configured in NetCash (`getNetCashBankAccounts`)
- Verified the customer's authentication / token to the bank feed provider is current
- Confirmed via the diagnostic queries in `diagnostic-queries.md` that the gap is on the ingestion side, not the matching side

If all of the above have been done and the gap persists, escalate.

### 5. NetSuite governance limit exhaustion

The user reports a "USAGE_LIMIT_EXCEEDED" or "SSS_USAGE_LIMIT_EXCEEDED" error on a NetCash operation. This typically indicates the script's governance budget is being consumed by a tenant-specific bottleneck (custom event scripts, heavy User Event scripts on transactions, or unusual record volumes). Netgain Support needs the tenant context to size the fix.

---

## Do NOT escalate the following

These map to known patterns and limitations — walk the diagnostic, don't escalate:

- **Single-subsidiary view shape** on `getNetCashReportMatches` — view-shape difference, not a tool failure; drop subsidiary breakdowns per `known-failure-modes.md` § Single-subsidiary view shape.
- **PSP variance** — `isCreateDifferenceJournal` + `offsetAccount` is the structured handling; not a defect.
- **FX rounding** — `isBookRounding` (or manual `isCreateDifferenceJournal` to FX gain/loss) is the structured handling; not a defect.
- **Authorized Date setup** for credit cards — configuration tweak; not a defect.
- **Entity-mapping merchant-only-search limitation** — documented limitation; the workaround (CONTAINS rule on description field) is the answer.
- **Cross-subsidiary "Entity not valid for subsidiary"** — NetSuite-level setup; user fixes via the entity's Subsidiary tab.

---

## What to put in the support ticket

When you recommend the user contact Netgain Support, draft the support-ticket payload for them so they don't have to gather it twice. The ticket should include:

### Required fields

- **Bank account name and internal ID.** Pull from `getNetCashBankAccounts`. Both fields, not just one.
- **Affected period or date range.** As specific as possible — "May 1-7, 2026" rather than "last month".
- **Reconciliation ID and period.** If the issue is on a specific reconciliation, pull the recon ID via `getNetCashReconciliations` and include the period (`Y-MM` format).
- **Affected transaction IDs.** The bank transaction internal IDs and / or GL transaction internal IDs the user is seeing the issue on. If the issue is multi-transaction, list up to 10 representative IDs and note the total count.
- **The exact error message.** Copy / paste the full text, including SSS error codes if any. "SSS_TIME_LIMIT_EXCEEDED" and "USAGE_LIMIT_EXCEEDED" are very different tickets.
- **Observed behavior vs. expected behavior.** "Auto-match did not run on May 5 deposits" is too vague; "Three deposits on May 5 totaling $48,200 were not auto-matched even though Active rule 'Stripe Daily Deposit' was configured for the account" is the right level.

### Useful additional context

- **What was already tried.** "Manual rule run executed; rule did fire but matched only 1 of 3 deposits." Saves the support engineer one round-trip.
- **Whether OneWorld.** Pull from the OneWorld detection rule in plugin-level `CLAUDE.md` (`ns_getSubsidiaries`, falling back to a `ns_runCustomSuiteQL` subsidiary count). Some support paths differ for OneWorld vs. single-subsidiary.
- **Bank feed provider.** Pull from `getNetCashBankAccounts`. The provider sometimes affects the troubleshooting path (e.g., re-import vs. re-trigger sync).
- **Subsidiary and currency** of the bank account if cross-currency or cross-subsidiary involvement is suspected.
- **Recent changes to the rule, mapping, or account configuration**, if known. "Rule was edited 2 days ago to add CONTAINS on description" is highly relevant.

### Format

Lead with one sentence stating the problem. Then bullet the fields above. Don't write a narrative — support engineers triage faster from structured tickets.

Example structure:

> **Issue:** Auto-match rule "Daily PSP Deposit" not firing on operating account.
> **Account:** Operating Checking (internal ID NNNN)
> **Period affected:** May 1-7, 2026
> **Bank transaction IDs:** NNNNN, NNNNN, NNNNN (3 of 3 deposits)
> **Error message:** None visible — transactions silently remain unmatched
> **Tried:** Manual rule execution (Active, Require Review OFF). Rule completed but 0 of 3 transactions matched.
> **Provider:** `<bank-feed-provider>` per NetCash configuration
> **OneWorld:** Yes (multiple subsidiaries)
> **Already considered:** Rule filter operator (CONTAINS) and entity mapping both verified — this is something else.

---

## Tone when recommending escalation

Surface escalation as the next step, not as a failure mode for the user. They've been troubleshooting; they want a path forward.

> "This crosses the threshold where Netgain Support needs to look at it directly — page-load timeouts on the reconciliation page typically indicate a tenant-level configuration that I can't see from here. Below is the support-ticket draft you can paste in. Want me to review it once you have a response?"

Don't loop back into more diagnostic walks after recommending escalation. The user has the support draft; let them open the ticket.
