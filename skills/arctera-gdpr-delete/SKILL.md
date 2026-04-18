---
name: arctera-gdpr-delete
description: Pre-deletion audit for GDPR Article 17 erasure requests. Identifies all records referencing a specific individual across all archives, classifies each by erasure eligibility (eligible, legal hold exempt, regulatory exempt, needs review), and produces an erasure scope report before any deletion is processed.
type: skill
---

# Skill: Arctera GDPR Erasure Pre-Deletion Audit

## Purpose
Before processing a GDPR Article 17 (Right to Erasure) or equivalent deletion request, identify the full scope of records referencing the individual and classify each by erasure eligibility. Records on legal hold, subject to regulatory retention requirements, or containing third-party data require separate handling. This audit ensures erasure is complete, legally compliant, and properly documented — preventing both under-deletion (compliance risk) and over-deletion (legal hold violation).

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Identify the Individual

Extract from the user's input:

| Identifier | Notes |
|---|---|
| Full name | Use both "First Last" and "Last, First" variants |
| Corporate email | Primary search signal |
| Personal email | Secondary if provided |
| Phone / employee ID | Additional identifiers if provided |
| Erasure basis | GDPR Article 17, CCPA, other — for the report |

If neither a name nor an email can be identified, stop and ask:
> *"Please provide the individual's full name and/or email address to scope the erasure audit."*

---

## Step 2 — Build Multi-Signal Queries

Same query set as DSAR:
```
FROMORTO:"individual@corpdomain.com"
ENTIREMESSAGE:"First Last"
ENTIREMESSAGE:"Last, First"
ENTIREMESSAGE:"individual@corpdomain.com"
```

Add personal email and phone variants if known.
Append date filter if provided: `AND MAILDATE:[YYYYMMDD TO YYYYMMDD]`

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute All Searches in Parallel

Call `search_email`, `search_file`, and `search_message` **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per archive.
De-duplicate results using ID fields.

---

## Step 4 — Classify Each Record by Erasure Eligibility

For every result, assign one of:

| Eligibility Status | Criteria |
|---|---|
| **Eligible — erase** | Contains only personal data of the subject, no legal basis to retain |
| **Exempt — Legal hold** | `isLegalHold: true` — cannot erase while hold is active |
| **Exempt — Regulatory retention** | Subject to SEC 17a-4, FINRA, or other mandatory retention |
| **Exempt — Legitimate interest** | Part of a business record with clear legitimate basis |
| **Partial — Third-party data** | Contains data about other individuals — subject's data may be redactable |
| **Needs review** | Cannot be automatically classified — requires DPO or legal assessment |

For **Legal hold** and **Regulatory exempt** items, call the preview tool to confirm before classifying — using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Erasure Scope Report

```
## Arctera GDPR Erasure Audit — [Individual Name]

**Individual:** [Full name and email]
**Erasure basis:** [GDPR Art. 17 / CCPA / other or "not specified"]
**Identifiers searched:** [list all used]
**Archives searched:** Email, Files, Collaboration
**Audit date:** [today's date — YYYY-MM-DD]

**Total records identified:** [N] ([n] emails, [n] files, [n] collab)
**Eligible for erasure:** [N]
**Exempt — Legal hold:** [N]
**Exempt — Regulatory retention:** [N]
**Exempt — Legitimate interest:** [N]
**Partial — Third-party data:** [N]
**Needs human review:** [N]

---

### Erasure Eligibility by Archive

| Archive | Total | Eligible | Legal Hold | Regulatory | Legitimate Interest | Needs Review |
|---------|-------|----------|------------|------------|---------------------|--------------|
| Email | N | N | N | N | N | N |
| Files | N | N | N | N | N | N |
| Collab | N | N | N | N | N | N |

---

### Records Eligible for Erasure

| # | Archive | Date | Subject / Filename / Room | Data Type | Reason Eligible |
|---|---------|------|--------------------------|-----------|-----------------|

---

### Records Exempt from Erasure

| # | Archive | Date | Subject / Filename / Room | Exemption Basis | Estimated Retention End |
|---|---------|------|--------------------------|-----------------|------------------------|

---

### Records Requiring Human Review

| # | Archive | Date | Subject / Filename / Room | Reason for Review |
|---|---------|------|--------------------------|-------------------|

---

### Records with Third-Party Data (Partial Erasure Candidates)

| # | Archive | Date | Subject / Filename / Room | Other Individual(s) | Recommendation |
|---|---------|------|--------------------------|---------------------|----------------|

---

### Recommended Actions

- [ ] Process erasure for [N] eligible records — no exemption applies
- [ ] **Do NOT erase** [N] records on legal hold until hold is released by legal counsel
- [ ] Confirm regulatory retention schedule for [N] exempt records with compliance team
- [ ] Submit [N] "needs review" records to DPO and legal counsel for classification
- [ ] For [N] partial records, assess whether redaction of subject's data is sufficient
- [ ] Log erasure action, date, basis, and exemptions in the data subject request register
- [ ] Re-run `/arctera-gdpr-delete` after erasure to confirm completion
```

---

## Behaviour Rules

- Run all searches **in parallel** across all three archives.
- De-duplicate results before classifying — same record may match multiple queries.
- **Never reproduce raw personal data values** — describe data type only.
- Do not classify a record as "Legal hold exempt" without confirming `isLegalHold: true` in the actual result or preview.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- This audit is advisory — actual erasure actions must be performed by the archive administrator, not by this skill.
