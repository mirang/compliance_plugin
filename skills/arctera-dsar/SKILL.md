---
name: arctera-dsar
description: Processes a Data Subject Access Request (DSAR) by finding all archived records referencing a named individual across email, files, and collaboration. Produces a personal data inventory categorised by data type, archive, and legal hold status — supporting GDPR, CCPA, and other privacy regulations.
type: skill
---

# Skill: Arctera Data Subject Access Request (DSAR)

## Purpose
Process a Data Subject Access Request by locating all records in the compliance archive that reference a specific individual. Searches across email, files, and collaboration using name variants, email addresses, and other identifiers. Produces a structured personal data inventory categorised by data type and archive, with legal hold status flagged. Supports GDPR Article 15, CCPA, and equivalent privacy rights frameworks.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Identify the Data Subject

Extract from the user's input:

| Identifier | How to find | Notes |
|---|---|---|
| Full name | Person's name (First Last) | Use both "First Last" and "Last, First" variants |
| Corporate email | `name@corpdomain.com` pattern | Primary search signal |
| Personal email | `name@gmail.com` etc. | Secondary signal if provided |
| Phone number | Any phone pattern | Format as `XXX-XXX-XXXX` and `(XXX) XXX-XXXX` |
| Employee ID | Any ID pattern after "employee ID:" | If provided |

If neither a name nor an email can be identified, stop and ask:
> *"Please provide the data subject's full name and/or email address to process this DSAR."*

---

## Step 2 — Build Multi-Signal Queries

Run all of the following queries across all three archives:

```
FROMORTO:"subject@corpdomain.com"
ENTIREMESSAGE:"First Last"
ENTIREMESSAGE:"Last, First"
ENTIREMESSAGE:"subject@corpdomain.com"
```

If personal email is known:
```
ENTIREMESSAGE:"personal@gmail.com"
FROMORTO:"personal@gmail.com"
```

If phone is known:
```
ENTIREMESSAGE:"XXX-XXX-XXXX"
ENTIREMESSAGE:"(XXX) XXX-XXXX"
```

Append date filter if provided: `AND MAILDATE:[YYYYMMDD TO YYYYMMDD]`

De-duplicate results across queries using the ID fields (`mailID`, `fileId`, `messageId`).

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute All Searches in Parallel

Call `search_email`, `search_file`, and `search_message` **in parallel** for each query variant:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per query per archive.

---

## Step 4 — Classify Each Record by Data Category

For every result, identify the personal data categories present:

| Data Category | Signals in snippet |
|---|---|
| Communications sent/received | Subject is sender or recipient |
| Contact information | Email, phone, address visible |
| Employment data | Job title, role, performance, HR references |
| Financial data | Salary, compensation, bank details |
| Health / medical | Medical references linked to the subject |
| Identity / credentials | SSN, DOB, passport, ID number |
| Files authored | Documents where subject is the author |
| Third-party references | Subject mentioned by others |

---

## Step 5 — Preview High-Relevance Items

For items where personal data is explicitly confirmed in the snippet, call the preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

Do not reproduce raw personal data values anywhere in the report. Describe the data type found, not the value.

---

## Step 6 — Produce the DSAR Response Package

```
## Arctera DSAR Report — [Data Subject Name]

**Data subject:** [Full name]
**Identifiers searched:** [email address(es), name variants, phone — as provided]
**Date range searched:** [range or "all available dates"]
**Archives searched:** Email, Files, Collaboration
**Request date:** [today's date — YYYY-MM-DD]

**Total records referencing this individual:** [N] ([n] emails, [n] files, [n] collab)
**Records with personal data confirmed:** [N]
**Records on legal hold:** [N]

---

### Personal Data Inventory by Category

| Data Category | Archive | Count | Earliest Record | Latest Record |
|---|---|---|---|---|
| Communications (sent) | Email | N | date | date |
| Communications (received) | Email | N | date | date |
| Employment data | File | N | date | date |
| Financial data | Email | N | date | date |
| Identity / credentials | Email | N | date | date |
| Files authored | Files | N | date | date |

---

### Records Containing Personal Data

| # | Archive | Date | Subject / Filename / Room | Data Category | Description (no raw values) |
|---|---------|------|--------------------------|---------------|----------------------------|

---

### Records on Legal Hold

| # | Archive | Date | Subject / Filename / Room | Hold Status | Note |
|---|---------|------|--------------------------|-------------|------|

---

### DSAR Compliance Summary

- **Total personal data records:** [N] across [N] archives
- **Legal hold exemptions:** [N] records — require legal review before disclosure
- **Third-party data:** [N] records also contain data about other individuals — may require redaction before disclosure
- **Recommended disclosure scope:** [N] records eligible for disclosure

---

### Recommended Next Steps

- [ ] Legal review of [N] items on legal hold before any disclosure
- [ ] Redact third-party personal data from [N] records before providing to subject
- [ ] Confirm DSAR response deadline and document in the data request register
- [ ] Run `/arctera-audit-pii` to check if additional PII categories were missed
- [ ] Log DSAR response with date, records disclosed, and any exemptions applied
```

---

## Behaviour Rules

- Search all three archives **in parallel** using all provided identifier variants.
- De-duplicate results before counting — same record may match multiple query variants.
- **Never reproduce raw personal data values** — always describe the data type.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- If zero records are found, state: *"No records referencing [name/email] found across [archives] for the specified scope."*
- Note: absence of records is itself a valid DSAR response element.
