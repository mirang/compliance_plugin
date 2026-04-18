---
name: arctera-aml
description: Anti-money laundering signal scan across all three Arctera archives. Detects structuring, suspicious activity references, beneficial ownership opacity, KYC/CDD gaps, unusual wire transfer discussions, and shell company references — producing a risk-tiered AML report.
type: skill
---

# Skill: Arctera Anti-Money Laundering Signal Scan

## Purpose
Scan the compliance archive for signals of anti-money laundering (AML) risk. Identifies explicit suspicious activity (SAR filings, structuring, smurfing), beneficial ownership opacity, shell company or nominee references, KYC/CDD gaps, unusual transaction structures, and complex layering arrangements. Produces a risk-tiered report for AML compliance teams, regulators, and internal investigations. Raw financial identifiers (account numbers, IBANs) are never reproduced — data types are described.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse AML Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Archive scope | "emails", "files", "chat", "all" | Default: all three |
| Date range | Year or date range | Optional |
| Custodians / counterparties | Email addresses or names | Optional |

If no archive keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for AML signals?"*

---

## Step 2 — Build AML Signal Queries

```
ENTIREMESSAGE:"money laundering" OR ENTIREMESSAGE:structuring OR ENTIREMESSAGE:"smurfing"
ENTIREMESSAGE:"suspicious activity" OR ENTIREMESSAGE:SAR OR ENTIREMESSAGE:"suspicious transaction report"
ENTIREMESSAGE:"wire transfer" OR ENTIREMESSAGE:"cash transaction" OR ENTIREMESSAGE:"cash deposit"
ENTIREMESSAGE:"beneficial owner" OR ENTIREMESSAGE:"ultimate beneficial owner" OR ENTIREMESSAGE:UBO
ENTIREMESSAGE:"shell company" OR ENTIREMESSAGE:"offshore account" OR ENTIREMESSAGE:nominee
ENTIREMESSAGE:"know your customer" OR ENTIREMESSAGE:KYC OR ENTIREMESSAGE:"customer due diligence" OR ENTIREMESSAGE:CDD
ENTIREMESSAGE:IBAN OR ENTIREMESSAGE:"routing number" OR ENTIREMESSAGE:SWIFT [in suspicious context]
ENTIREMESSAGE:layering OR ENTIREMESSAGE:placement [in financial transaction context]
```

Append custodian (`FROMORTO:"email"`) and date (`MAILDATE:[YYYYMMDD TO YYYYMMDD]`) filters with AND where provided.

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call all applicable search tools **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per query.

---

## Step 4 — Classify Each Result

**AML Signal Type:**
| Type | Criteria |
|---|---|
| Structuring / smurfing | Breaking up transactions to avoid reporting thresholds |
| Suspicious activity | Explicit SAR filing reference, suspicion flagged internally |
| Beneficial ownership | UBO opacity, shell company, nominee structure discussed |
| KYC / CDD gap | Due diligence concern, missing customer information, gap identified |
| Unusual transaction | Large or unusual wire transfers, cash transactions in suspicious context |
| Layering | Complex multi-entity transaction structure to obscure origin of funds |

**Risk Level:**
| Level | Criteria |
|---|---|
| Critical | Explicit SAR filing, structuring reference, or confirmed money laundering discussion |
| High | Beneficial ownership opacity or unusual transaction structure with suspicious context |
| Medium | KYC/CDD gap or due diligence concern without clear resolution |
| Low | General AML compliance reference (policy, training, routine due diligence) |

For **Critical** and **High** items, call the preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the AML Risk Report

```
## Arctera AML Signal Report

**Custodians / counterparties in scope:** [list or "all"]
**Date range:** [range or "all dates"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Total records found:** [N] ([n] emails, [n] files, [n] collab)
**Critical:** [N] | **High:** [N] | **Medium:** [N] | **Low:** [N]

---

### Critical Items — SAR, Structuring, or Explicit AML Reference

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-------------|

---

### High-Risk Items — Beneficial Ownership or Transaction Structure Concerns

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|---------|

---

### Medium Items — KYC/CDD Gaps or Due Diligence Concerns

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|---------|

---

### AML Risk Narrative

[2–3 paragraphs: nature of AML risk, key parties and counterparties, transaction patterns,
whether SAR obligations may be triggered, recommended escalation path]

---

### Recommended Actions

- [ ] Escalate Critical items to Chief Compliance Officer and AML team immediately
- [ ] Assess SAR filing obligations for Critical items with compliance team
- [ ] Preserve all flagged items — run `/arctera-legal-hold`
- [ ] Run `/arctera-custodian` on parties involved in suspicious transactions
- [ ] Engage outside AML counsel for Critical and High-risk items
- [ ] Review KYC/CDD files for counterparties identified in Medium items
```

---

## Behaviour Rules

- Run all applicable search tools **in parallel**.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- **Never reproduce raw financial identifiers** — account numbers, IBANs, routing numbers, SWIFT codes — describe the data type only.
- Do not call more than 10 get tools without checking with the user.
- If zero results are found, state: *"No AML signals detected across [N] records in [scope]."*
