---
name: arctera-audit-pii
description: Automated PII detection audit over the Arctera archive. Scans archived emails, files, and collaboration messages for sensitive data patterns and produces a flagged redaction report.
---

# Skill: Arctera PII Audit & Sensitive Data Scrubbing

## Purpose
Scan archived emails, files, and collaboration messages in the Arctera compliance archive for
Personally Identifiable Information (PII) and other sensitive data. Preview flagged items using
the dedicated get tools to confirm the presence of PII, then produce a structured redaction report.

---

## Available Tools

| Tool            | Archive        | Preview tool  | Key ID fields returned                          |
|-----------------|---------------|---------------|------------------------------------------------|
| `search_email`  | Emails         | `get_email`   | `mailID`                                        |
| `search_file`   | Files/Docs     | `get_file`    | `fileId`, `batchId`, `messageId`               |
| `search_message`| Collab/Chat    | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date`  |

---

## PII Category Reference

| Category                  | Search signals                                                       |
|---------------------------|----------------------------------------------------------------------|
| Social Security Number     | "social security", "SSN", "tax ID"                                  |
| Credit / Debit Card        | "credit card", "card number", "CVV", "debit card"                  |
| Home / Personal Address    | "home address", "personal address", "residential address"           |
| Date of Birth              | "date of birth", "DOB", "born on"                                   |
| Passport / National ID     | "passport number", "national ID", "driver's license", "driving licence" |
| Bank Account / Routing     | "account number", "routing number", "IBAN", "SWIFT", "sort code"   |
| Medical / Health Info      | "medical record", "diagnosis", "prescription", "health condition"   |
| Salary / Compensation      | "salary", "W-2", "pay stub", "payslip", "annual compensation"       |
| Personal Contact Info      | Personal (non-work) email addresses, personal mobile numbers        |

---

## Step 1 — Resolve Archive Scope

Determine which archives to scan based on the user's exact wording:

| Keyword(s) in prompt                                                              | Action                                        |
|-----------------------------------------------------------------------------------|-----------------------------------------------|
| "email", "mail", "mails"                                                          | `search_email` only                           |
| "file", "files", "document", "documents", "attachment", "spreadsheet", "pdf"      | `search_file` only                            |
| "chat", "collab", "collaboration", "teams", "slack", "bloomberg chat", "im"       | `search_message` only                         |
| "both", "all", "everything", "full audit", "all archives"                         | All three tools in parallel                   |
| **None of the above keywords present**                                            | **STOP — ask the user before calling anything** |

When asking:
> *"Would you like to scan emails, files, collaboration messages, or all three for PII?"*

If the user has provided additional filters (date range, sender, project name), append them to every PII query with `AND`.

---

## Step 2 — Build PII-Targeted Lucene Queries

Run each of the following queries against the applicable archives. Where the user supplied an
additional filter (e.g. `MAILDATE:[20230101 TO 20231231]` or `SENDER:firm*`), append it with `AND`
to every query below.

```
ENTIREMESSAGE:"social security" OR ENTIREMESSAGE:SSN
ENTIREMESSAGE:"date of birth" OR ENTIREMESSAGE:DOB
ENTIREMESSAGE:"credit card" OR ENTIREMESSAGE:"card number" OR ENTIREMESSAGE:CVV
ENTIREMESSAGE:"home address" OR ENTIREMESSAGE:"personal address"
ENTIREMESSAGE:"passport number" OR ENTIREMESSAGE:"driver's license"
ENTIREMESSAGE:"account number" OR ENTIREMESSAGE:"routing number" OR ENTIREMESSAGE:IBAN
ENTIREMESSAGE:"medical record" OR ENTIREMESSAGE:diagnosis OR ENTIREMESSAGE:prescription
ENTIREMESSAGE:"W-2" OR ENTIREMESSAGE:"pay stub" OR ENTIREMESSAGE:salary
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID
- Do not add wildcards to quoted phrases — they already perform phrase matching.

---

## Step 3 — Execute PII Searches

For each query, call the relevant search tool(s) with:
- `contentDetails: 1` (snippets) — initial scan pass
- `pageSize: 20`
- `pageNum: 0` for `search_email` (0-indexed); `pageNum: 1` for `search_file` and `search_message` (1-indexed)

Call searches **in parallel** where multiple archives are in scope.
Run at most 10 queries before checking in with the user if the result set is very large.

---

## Step 4 — Classify Each Flagged Item

For every result returned, assign:

**PII Category** — which type from the reference table above.

**Confidence:**
| Level  | Criteria                                                                          |
|--------|-----------------------------------------------------------------------------------|
| High   | Explicit PII term with surrounding context visible in the snippet                |
| Medium | PII-adjacent language (e.g. "compensation discussion") — needs human confirmation |
| Low    | Keyword match only, could be generic reference                                   |

**Redaction Urgency:**
| Level    | Meaning                                                         |
|----------|-----------------------------------------------------------------|
| Critical | Clear PII pattern confirmed in snippet — must redact before export |
| Review   | Potential PII — requires a human to confirm before redacting   |

For all **High-confidence Critical** items, call the appropriate get tool to confirm PII in the full content:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

Only call get tools using IDs obtained in this session — never reuse IDs from prior conversations.

---

## Step 5 — Produce the PII Audit Report

Output a structured report using this format:

```
## Arctera PII Audit Report

**Scan scope:** [Emails / Files / Collab / All]
**User filter applied:** [date range / sender / project — or "none"]
**PII queries run:** [N]
**Total records scanned:** [N] ([n] emails, [n] files, [n] collab)
**Items flagged — Critical:** [N]
**Items flagged — Review:** [N]

---

### Critical Items — Must Redact Before Export

| # | Archive | Date | Sender / Author | Subject / Filename / Chat Room | PII Category | Confidence | Masked Excerpt |
|---|---------|------|-----------------|-------------------------------|--------------|------------|----------------|
| 1 | Email   | ...  | ...             | ...                           | SSN          | High       | "...SSN: XXX-XX-####..." |

---

### Review Items — Requires Human Confirmation

| # | Archive | Date | Sender / Author | Subject / Filename / Chat Room | PII Category | Confidence | Excerpt |
|---|---------|------|-----------------|-------------------------------|--------------|------------|---------|
| 2 | File    | ...  | ...             | ...                           | Salary/Comp  | Medium     | "...annual comp package..." |

---

### PII Summary by Category

| PII Category      | Count | Highest Confidence |
|-------------------|-------|--------------------|
| SSN               | N     | High               |
| Credit Card       | N     | Medium             |
| Home Address      | N     | High               |

---

### Recommended Redaction Actions

- [ ] Flag item #N (archive: Email, mailID: ...) — contains SSN — redact before export
- [ ] Hold production set export until Critical items are cleared
- [ ] Escalate Critical items to redaction team
- [ ] Re-run scan after redaction to confirm clearance
```

---

## Masking Rule

Never reproduce raw PII values in the report. Use masked format:
- SSN: `XXX-XX-####`
- Credit card: `XXXX-XXXX-XXXX-####`
- Phone: `XXX-XXX-####`
- Address: describe as "[home address detected]"
- All others: describe the PII type, not the value

---

## Behaviour Rules

- Call all applicable search tools **in parallel**, not sequentially.
- Only call `get_email` / `get_file` / `get_message` using IDs from the **current session**.
- If zero PII items found, state: *"No PII patterns detected across [N] records scanned in [scope]."*
- Do not run more than 10 queries without checking with the user — large audits should be approved.
- Do not reproduce raw PII values anywhere — always mask.
- For `get_file` results, `html_content` is rendered HTML — scan it for PII patterns and summarise findings; do not display raw HTML.
- For `get_message` results, check the `text` field of each message in the returned `collab_messages` list.
