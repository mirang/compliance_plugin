# Arctera Compliance Suite — Claude Code Plugin

> End-to-end e-discovery, PII auditing, regulatory compliance, and legal review workflows powered by the Arctera compliance archive.

---

## Overview

The **Arctera Compliance Suite** connects Claude Code directly to your Arctera compliance archive, enabling legal, compliance, and e-discovery teams to run sophisticated multi-archive investigations using plain language — no query expertise required.

Claude translates your requests into precise Lucene queries, searches emails, files, and collaboration messages in parallel, triages the most relevant results, and delivers structured reports ready for legal review, regulatory response, or export clearance.

---

## Requirements

- **Claude Code** (desktop, web, or IDE extension)
- **Arctera MCP server** configured and connected (`arctera-compliance` MCP server)
- Access to the Arctera compliance archive

---

## Installation

```bash
claude --plugin-dir /path/to/arctera-compliance
```

Or add to your Claude Code settings:

```json
{
  "plugins": ["/path/to/arctera-compliance"]
}
```

---

## Commands

### E-Discovery & Legal

| Command | Description | Example |
|---|---|---|
| `/arctera-review` | First-pass legal review across any archive | `/arctera-review emails about Project Falcon from last year` |
| `/arctera-matter` | Full scoped review for a named legal matter | `/arctera-matter matter: SEC Investigation, parties: john.doe@enron.com, date: 2001-2002` |
| `/arctera-custodian` | All communications to/from a specific person | `/arctera-custodian john.doe@enron.com from 2001-01-01 to 2002-01-01` |
| `/arctera-timeline` | Chronological event timeline around a topic | `/arctera-timeline Project Raptor between ken.lay@enron.com and jeff.skilling@enron.com` |
| `/arctera-thread` | Reconstruct a full cross-archive conversation | `/arctera-thread "mark-to-market" involving andrew.fastow@enron.com` |
| `/arctera-legal-hold` | Identify legal hold candidates | `/arctera-legal-hold matter: Enron Litigation, parties: jeff.skilling@enron.com, date: 2001` |

### PII & Privacy

| Command | Description | Example |
|---|---|---|
| `/arctera-audit-pii` | PII detection and redaction report | `/arctera-audit-pii 20 emails from enron.com, PII: SSN and credit card` |
| `/arctera-dsar` | Data Subject Access Request processing | `/arctera-dsar individual: Jane Smith, email: jane.smith@enron.com` |
| `/arctera-gdpr-delete` | Pre-erasure audit before GDPR deletion | `/arctera-gdpr-delete individual: John Doe, email: john.doe@enron.com` |
| `/arctera-data-exposure` | Sensitive data sent externally | `/arctera-data-exposure from enron.com to external domains, date: 2001` |

### Regulatory & Conduct Risk

| Command | Description | Example |
|---|---|---|
| `/arctera-regulatory` | SEC, FINRA, FCA, CFTC red flag scan | `/arctera-regulatory emails, regulator: SEC, date: 2001` |
| `/arctera-insider-risk` | Insider trading and MNPI detection | `/arctera-insider-risk all archives, tickers: ENE, date: 2001-09 to 2001-12` |
| `/arctera-fcpa` | FCPA and anti-bribery scan | `/arctera-fcpa all archives from 2000 to 2002` |
| `/arctera-conflicts` | Conflicts of interest detection | `/arctera-conflicts emails involving andrew.fastow@enron.com` |
| `/arctera-aml` | Anti-money laundering signal scan | `/arctera-aml all archives, date: 2001` |

### Cross-Archive & Correlation

| Command | Description | Example |
|---|---|---|
| `/arctera-cross-archive` | Correlate a topic across all three archives | `/arctera-cross-archive "special purpose entities" from 2000 to 2002` |
| `/arctera-project` | All communications about a deal or project | `/arctera-project Project LJM from 1999 to 2001` |
| `/arctera-attachment-audit` | Attachment inventory by type and sender | `/arctera-attachment-audit xlsx files from finance@enron.com in 2001` |

### People & Communication Patterns

| Command | Description | Example |
|---|---|---|
| `/arctera-between` | All communications between two people | `/arctera-between ken.lay@enron.com and jeff.skilling@enron.com in 2001` |
| `/arctera-external` | Outbound communications to external domains | `/arctera-external from enron.com, date: 2001, filter: confidential` |
| `/arctera-channel-audit` | Audit a specific messaging channel | `/arctera-channel-audit Bloomberg channel for jeff.skilling@enron.com in 2001` |

### Reporting & Export Readiness

| Command | Description | Example |
|---|---|---|
| `/arctera-pre-export` | Pre-export PII, hold, and privilege check | `/arctera-pre-export emails about Project Raptor from 2001` |
| `/arctera-summary` | Executive summary for non-legal stakeholders | `/arctera-summary Enron SEC investigation key communications 2001` |

---

## Archive Coverage

Each command intelligently routes to the correct Arctera archive(s):

| Archive | Contents | Tool |
|---|---|---|
| **Email** | Inbound/outbound email, Bloomberg/Reuters messages | `search_email` + `get_email` |
| **Files** | Documents, spreadsheets, PDFs, attachments | `search_file` + `get_file` |
| **Collaboration** | Teams, Slack, Bloomberg Chat, instant messages | `search_message` + `get_message` |

---

## Key Capabilities

- **Natural language → Lucene**: No query syntax knowledge required
- **Parallel multi-archive search**: All three archives searched simultaneously
- **Intelligent triage**: Top results automatically previewed in full
- **PII masking**: Raw PII values never reproduced — always masked in reports
- **Structured reports**: Every command produces a consistent, audit-ready report
- **Session safety**: Archive IDs never reused across sessions

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.

---

## License

MIT — see LICENSE for details.
