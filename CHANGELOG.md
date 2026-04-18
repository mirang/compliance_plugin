# Changelog — Arctera Compliance Suite

All notable changes to this plugin are documented here.

---

## [2.0.0] — 2026-04-08

### Added — 21 new commands and skills

**E-Discovery & Legal**
- `/arctera-matter` — scoped legal matter review with party and date range filtering
- `/arctera-custodian` — full custodian communication pull across all archives
- `/arctera-timeline` — chronological event timeline reconstruction across archives
- `/arctera-thread` — cross-archive conversation thread reconstruction
- `/arctera-legal-hold` — legal hold candidate identification and reporting

**PII & Privacy**
- `/arctera-dsar` — Data Subject Access Request processing (GDPR, CCPA)
- `/arctera-gdpr-delete` — pre-erasure audit before GDPR Article 17 deletion
- `/arctera-data-exposure` — sensitive data sent to external or unexpected recipients

**Regulatory & Conduct Risk**
- `/arctera-regulatory` — SEC, FINRA, FCA, CFTC red flag scan
- `/arctera-insider-risk` — insider trading and MNPI signal detection
- `/arctera-fcpa` — FCPA and anti-bribery compliance scan
- `/arctera-conflicts` — conflicts of interest detection
- `/arctera-aml` — anti-money laundering signal scan

**Cross-Archive & Correlation**
- `/arctera-cross-archive` — topic correlation across email, files, and collaboration
- `/arctera-project` — full project or deal communication pull
- `/arctera-attachment-audit` — attachment inventory by type, sender, or recipient

**People & Communication Patterns**
- `/arctera-between` — point-to-point communication audit between two parties
- `/arctera-external` — outbound communication audit to external domains
- `/arctera-channel-audit` — channel-specific audit (Bloomberg, Teams, Slack, Reuters)

**Reporting & Export Readiness**
- `/arctera-pre-export` — pre-export PII, legal hold, and privilege clearance check
- `/arctera-summary` — executive summary report for non-legal stakeholders

### Changed
- `manifest.json` bumped to version `2.0.0`
- Added `displayName`, `author`, `license`, `keywords` fields to manifest

---

## [1.0.0] — Initial Release

### Added
- `/arctera-review` — first-pass legal and compliance review
- `/arctera-audit-pii` — PII audit and sensitive data scrubbing
- `PostToolUse` hook for all Arctera MCP tool calls
