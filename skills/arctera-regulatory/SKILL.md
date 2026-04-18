---
name: arctera-regulatory
description: Scans all three Arctera archives for regulatory red flags — SEC, FINRA, FCA, CFTC, DOJ references, subpoenas, enforcement actions, Wells notices, and whistleblower activity. Produces a risk-tiered regulatory exposure report.
type: skill
---

# Skill: Arctera Regulatory Red Flag Scan

## Purpose
Identify communications and documents in the compliance archive that indicate actual or anticipated regulatory exposure. Covers active regulatory contact (subpoenas, examination letters, Wells notices), enforcement actions (consent orders, settlements), internal awareness of regulatory risk, and whistleblower activity. Produces a tiered report distinguishing active regulatory contact from internal risk discussion.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Regulatory Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Specific regulator | Named regulator (SEC, FINRA, FCA, CFTC, DOJ, FERC, OCC) | Prioritise this regulator's query |
| Archive scope | "emails", "files", "chat", "all" | Default: all three |
| Date range | Year or date range references | Optional |
| Custodians | Email addresses or names | Optional |

If no archive keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for regulatory red flags?"*

---

## Step 2 — Build Regulatory Signal Queries

Run all of the following queries against the applicable archives:

```
ENTIREMESSAGE:SEC OR ENTIREMESSAGE:FINRA OR ENTIREMESSAGE:CFTC OR ENTIREMESSAGE:FCA OR ENTIREMESSAGE:DOJ OR ENTIREMESSAGE:FERC OR ENTIREMESSAGE:OCC
ENTIREMESSAGE:subpoena OR ENTIREMESSAGE:"regulatory inquiry" OR ENTIREMESSAGE:"government investigation"
ENTIREMESSAGE:"cease and desist" OR ENTIREMESSAGE:"Wells notice" OR ENTIREMESSAGE:"enforcement action"
ENTIREMESSAGE:"examination" OR ENTIREMESSAGE:"regulatory review" OR ENTIREMESSAGE:"regulatory audit"
ENTIREMESSAGE:"consent order" OR ENTIREMESSAGE:"deferred prosecution" OR ENTIREMESSAGE:"non-prosecution agreement"
ENTIREMESSAGE:"whistleblower" OR ENTIREMESSAGE:"qui tam" OR ENTIREMESSAGE:"protected disclosure"
ENTIREMESSAGE:settlement AND (ENTIREMESSAGE:SEC OR ENTIREMESSAGE:FINRA OR ENTIREMESSAGE:regulatory)
```

If a specific regulator is named, run that regulator's query **first** before running the full sweep.
Append custodian (`FROMORTO:"email"`) and date (`MAILDATE:[YYYYMMDD TO YYYYMMDD]`) filters with AND where provided.

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call all applicable search tools **in parallel** for each query:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per query per archive.
Do not run more than 10 queries without checking with the user.

---

## Step 4 — Classify Each Result

**Regulatory Signal Type:**
| Signal Type | Criteria |
|---|---|
| Active inquiry | Subpoena received, investigation notice, examination letter, Wells notice |
| Enforcement action | Consent order, settlement, deferred prosecution, formal charges |
| Internal awareness | Internal discussion of actual or anticipated regulatory investigation |
| Whistleblower | Mention of whistleblower, qui tam, protected disclosure |
| General compliance | Routine compliance review, audit, or examination reference |

**Risk Level:**
| Level | Criteria |
|---|---|
| Critical | Active regulator contact confirmed in snippet (subpoena, Wells notice, enforcement) |
| High | Internal discussion of actual or anticipated regulatory inquiry |
| Medium | General regulatory term in a context suggesting awareness of risk |
| Low | Routine compliance/audit reference with no surrounding risk context |

For **Critical** and **High** items, call the appropriate preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Regulatory Risk Report

```
## Arctera Regulatory Red Flag Report

**Regulator focus:** [specific regulator or "all regulators"]
**Date range:** [range or "all dates"]
**Custodians in scope:** [list or "all"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Total records found:** [N] ([n] emails, [n] files, [n] collab)
**Critical:** [N] | **High:** [N] | **Medium:** [N] | **Low:** [N]

---

### Critical Items — Active Regulatory Contact or Enforcement

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Regulator | Signal Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-----------|-------------|-------------|

---

### High-Risk Items — Internal Regulatory Risk Discussion

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Regulator | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-----------|-------------|---------|

---

### Medium Items — General Regulatory Awareness

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Regulator | Excerpt |
|---|---------|------|-----------------|--------------------------|-----------|---------|

---

### Regulatory Exposure Narrative

[2–3 paragraphs: nature and scope of regulatory exposure, regulators involved, key parties,
timeline of regulatory contact, and recommended escalation path]

---

### Recommended Next Steps

- [ ] Escalate Critical items to General Counsel and outside counsel immediately
- [ ] Preserve all Critical and High-risk items — run `/arctera-legal-hold`
- [ ] Run `/arctera-custodian` on key parties named in regulatory communications
- [ ] Run `/arctera-audit-pii` before any regulatory production set is exported
- [ ] Engage outside counsel with [regulator] expertise for advisory
- [ ] Review privilege status of all attorney communications in flagged items
```

---

## Behaviour Rules

- Run all applicable search tools **in parallel**.
- If a specific regulator is named, prioritise that regulator's query first.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- Do not call more than 10 get tools without checking with the user.
- Do not run more than 10 queries without checking with the user.
- If zero results are found, state: *"No regulatory red flags detected across [N] records in [scope]."*
