You are running the **Arctera Data Subject Access Request (DSAR)** workflow.

User request: $ARGUMENTS

---

## 1 — Identify the data subject from $ARGUMENTS

Extract:
- **Full name** — first and last name (e.g. "Jane Smith")
- **Email address** — corporate or personal (e.g. `jane.smith@enron.com` or `jsmith@gmail.com`)
- **Additional identifiers** — phone number, employee ID, or any other identifiers mentioned
- **Date range** — optional scope for the DSAR (defaults to all available dates)

If neither a name nor an email address can be identified, stop and ask:
> *"Please provide the data subject's full name and/or email address to process this DSAR."*

---

## 2 — Build multi-signal Lucene queries

Run all of the following queries across all three archives:

```
FROMORTO:"subject@domain.com"
ENTIREMESSAGE:"First Last"
ENTIREMESSAGE:"Last, First"
ENTIREMESSAGE:"subject@domain.com"
```

If a personal email is known:
```
ENTIREMESSAGE:"personal@gmail.com"
FROMORTO:"personal@gmail.com"
```

If a phone number is known (format variants):
```
ENTIREMESSAGE:"XXX-XXX-XXXX"
ENTIREMESSAGE:"(XXX) XXX-XXXX"
```

Append date filter to all queries if provided: `AND MAILDATE:[YYYYMMDD TO YYYYMMDD]`

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all three archives in parallel

Call `search_email`, `search_file`, and `search_message` with each query simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 total results per archive.
De-duplicate results across queries (same `mailID` / `fileId` / `messageId`).

---

## 4 — Classify results by data category

For each result, identify what personal data it contains:

| Data Category | Signals |
|---|---|
| Contact information | Email address, phone, address |
| Employment data | Job title, salary, performance, HR records |
| Financial data | Bank details, payroll, compensation |
| Health / medical | Any medical references |
| Identity / credentials | SSN, date of birth, passport, ID |
| Communications | Emails/chats sent or received by the subject |
| Files / documents | Files authored, shared, or referencing the subject |

---

## 5 — Preview high-relevance items

For items where the data subject's personal data appears explicitly in the snippet, call:
- Email → `get_email(mailID)`
- File → `get_file(fileId, batchId, messageId)`
- Collab → `get_message(chatRoomId, collabMessageId, chatRoomName, matterId, collabDate)`

Only use IDs from this session. Do not reproduce raw PII — describe the data type found.

---

## 6 — Deliver the DSAR Response Package

```
## Arctera DSAR Report — [Data Subject Name]

**Data subject:** [Full name]
**Identifiers used:** [email, name variants, phone — as provided]
**Date range searched:** [range or "all available dates"]
**Archives searched:** Email, Files, Collaboration
**Request date:** [today's date]

**Total records referencing this individual:** [N] ([n] emails, [n] files, [n] collab)
**Records with personal data confirmed:** [N]

---

### Personal Data Inventory by Category

| Data Category | Archive | Count | Earliest | Latest |
|---|---|---|---|---|
| Communications (sent) | Email | N | date | date |
| Communications (received) | Email | N | date | date |
| Employment data | File | N | date | date |
| Identity / credentials | Email | N | date | date |

---

### Records Containing Personal Data

| # | Archive | Date | Subject / Filename / Room | Data Category | Description |
|---|---------|------|--------------------------|---------------|-------------|
*(No raw PII values — describe data type only)*

---

### Records on Legal Hold

| # | Archive | Date | Subject | Hold Status |
|---|---------|------|---------|-------------|

---

### DSAR Compliance Notes

- **Right to access**: [N] records identified across [N] archives
- **Legal hold exemption**: [N] records on hold — may require legal review before disclosure
- **Retention period**: [Note any records past standard retention if identifiable]
- **Third-party data**: [Note any records containing data about other individuals]

---

### Recommended Next Steps

- [ ] Legal review of [N] items on legal hold before disclosure
- [ ] Confirm scope with data subject — are all identifiers covered?
- [ ] Run `/arctera-audit-pii` to confirm no additional PII categories were missed
- [ ] Document DSAR response date and records disclosed for compliance log
```

**Masking rule:** Never reproduce raw PII values in the report. Describe the type of data found, not the value.
