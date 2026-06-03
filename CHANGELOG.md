# Changelog

All notable changes to the NetCash Cowork plugin are documented here. The format is based on [Keep a Changelog](https://keepachangelog.com/), and the project adheres to [Semantic Versioning](https://semver.org/).

## 1.0.0 — 2026-06-03

Initial public release. NetCash Cowork is an AI copilot for **NetCash** — Netgain's NetSuite-native SuiteApp for bank reconciliation and cash management — bringing matching, automation, dashboards, reporting, and troubleshooting into a conversational, controller-in-the-chair workflow that operates on your existing NetSuite data.

### Skills

- **Matching** (`/find-matches`) — investigate unmatched bank transactions and find 1:1 GL matches, cash applications to open invoices and bills, and grouped matches.
- **Automation rules** (`/suggest-rules`) — analyze manual match history and propose or create GL Match, Create Transaction, Create Transfer, and Cash Application rules.
- **Reconciliation dashboards** (`/cash-position`, `/recon-status`, `/daily-check`, `/exception-summary`) — Account and GL balances across bank and credit-card accounts, per-account reconciliation progress, and exceptions ranked by financial impact.
- **Ad-hoc reporting** (`/investigate-variance`, plus free-form queries) — natural-language questions over bank activity, GL activity, and reconciliations.
- **Pattern insights** (`/insights`) — proactive surfacing of recurring spend, payment-processor fee patterns, vendor concentration, and month-over-month anomalies.
- **Troubleshooting** (`/troubleshoot`) — guided, named-failure-mode diagnostics for common reconciliation issues.

### Notes

- All write actions (submitting a match, creating a rule, applying a payment) require explicit confirmation.
- Requires the NetCash SuiteApp and a configured NetSuite MCP connection — see [GETTING-STARTED.md](GETTING-STARTED.md) and [CONNECTORS.md](CONNECTORS.md).
