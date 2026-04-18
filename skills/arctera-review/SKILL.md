---
name: arctera-review
description: Automated first-pass legal and compliance review over the Arctera archive. Searches emails, files, and collaboration messages, previews the most relevant items, and produces a structured risk report.
---

# Skill: Arctera First-Pass Legal & Compliance Review

## Purpose
Perform an automated first-pass legal and compliance review over the Arctera compliance archive.
This skill searches emails, files, and/or collaboration messages, previews the most relevant items
using the dedicated get tools, and produces a structured risk report — without the user having to
manually open each item.

---

## Available Tools

| Tool            | Archive        | Preview tool  | Key ID fields returned                          |
|-----------------|---------------|---------------|------------------------------------------------|
| `search_email`  | Emails         | `get_email`   | `mailID`                                        |
| `search_file`   | Files/Docs     | `get_file`    | `fileId`, `batchId`, `messageId`               |
| `search_message`| Collab/Chat    | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date`  |

---

## Step 1 — Resolve Archive Scope

Determine which archives to search based on the user's exact wording:

| Keyword(s) in prompt                                                              | Action                                        |
|-----------------------------------------------------------------------------------|-----------------------------------------------|
| "email", "mail", "mails"                                                          | `search_email` only                           |
| "file", "files", "document", "documents", "attachment", "spreadsheet", "pdf"      | `search_file` only                            |
| "chat", "collab", "collaboration", "teams", "slack", "bloomberg chat", "im"       | `search_message` only                         |
| "both", "all", "everything", "all archives", "compliance search"                  | All three tools in parallel                   |
| **None of the above keywords present**                                            | **STOP — ask the user before calling anything** |

When asking:
> *"Would you like to search emails, files, collaboration messages, or all three?"*

---

## Step 2 — Build Lucene Queries

Translate the user's natural language request into Lucene query strings.

### Field reference
- `ENTIREMESSAGE:` — full-text across body, subject, and attachments
- `SUBJECT:` — subject/topic
- `SENDER:` / `FROMORTO:` — sender or any party
- `MAILDATE:[YYYYMMDD TO YYYYMMDD]` — date range
- `ATTACHMENTS.FILENAME:` / `ATTACHMENTS.EXTENSION:` — attachment filter
- `CLASSIFICATION.TAGS:` — compliance tags already applied
- `SOURCETYPE:` — channel (email, bloomberg, reuters, teams, slack)

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID
- For domain searches ("from enron.com"): use `SENDER:enron*` or `FROMORTO:"enron.com"` — NOT `SENDER:*@enron.com`

### Example translations
| User says                                               | Lucene query                                                        |
|---------------------------------------------------------|---------------------------------------------------------------------|
| "emails from enron.com about Project Phoenix"           | `SENDER:enron* AND ENTIREMESSAGE:"Project Phoenix"`                |
| "collab chats from last year mentioning confidentiality"| `ENTIREMESSAGE:confidential* AND MAILDATE:[20250101 TO 20251231]`  |
| "files related to the 2023 audit"                       | `ENTIREMESSAGE:audit* AND MAILDATE:[20230101 TO 20231231]`         |
| "any mention of NDA or non-disclosure"                  | `ENTIREMESSAGE:NDA OR ENTIREMESSAGE:"non-disclosure"`              |

---

## Step 3 — Execute Searches

Call the relevant tools with:
- `contentDetails: 1` (snippets) — for the initial triage pass
- `pageSize: 20`
- `pageNum: 0` for `search_email` (0-indexed); `pageNum: 1` for `search_file` and `search_message` (1-indexed)

When searching all three archives, call `search_email`, `search_file`, and `search_message` **in parallel** with the same query.

Paginate using `pageNum` (increment by 1 each page) while `has_more` is `true`, up to a maximum of 100 total results. If the cap is reached, notify the user and offer to narrow the scope.

---

## Step 4 — Triage and Prioritise

From all search results combined, identify the top items by:

1. Snippet directly references all or most of the user's key terms
2. Items with attachments (`ATTCOUNT > 0` for emails) or flagged as significant
3. Items with `CLASSIFICATION.TAGS` already applied by the compliance system
4. Items involving sensitive parties (executives, external counsel, regulators)

Select the **top 10 most relevant** items for full preview. For each:
- Emails → call `get_email(mailID: <mailID>)`
- Files → call `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → call `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

Only call get tools for IDs obtained in this session — never reuse IDs from prior conversations.

---

## Step 5 — Identify Legal Risks

For each previewed item, check for the following risk categories:

| Risk Category          | Signals to look for                                                         |
|------------------------|-----------------------------------------------------------------------------|
| Confidentiality breach | "confidential", "NDA", "do not share", "proprietary", "under seal"          |
| Agreement terms        | "agreement", "clause", "obligation", "breach", "penalty", "termination"     |
| Financial exposure     | "settlement", "liability", "damages", "payment", "bonus", "compensation"    |
| Regulatory concern     | "SEC", "FINRA", "audit", "compliance", "regulatory", "investigation"        |
| Communication risk     | Hidden/BCC'd parties, external counsel cc'd, unusual sender-recipient pairs |
| Legal hold relevance   | `isLegalHold: true` on collab items; `CLASSIFICATION.TAGS:flagged`         |

---

## Step 6 — Produce the Risk Report

Output a structured report using this format:

```
## Arctera Compliance Review — First-Pass Report

**Search scope:** [Emails / Files / Collab / All]
**Archives searched:** search_email, search_file, search_message (as applicable)
**Query:** `[Lucene query used]`
**Total records found:** [N] ([n] emails, [n] files, [n] collab)
**Records reviewed:** [N]
**Items previewed in full:** [N]
**High-risk items identified:** [N]

---

### High-Risk Items

| # | Archive | Date | Sender / Author | Subject / Filename / Chat Room | Risk Category | Key Excerpt |
|---|---------|------|-----------------|-------------------------------|---------------|-------------|
| 1 | Email   | ...  | ...             | ...                           | Confidentiality | "...do not share..." |
| 2 | File    | ...  | ...             | ...                           | Financial exposure | "...settlement of $..." |
| 3 | Collab  | ...  | ...             | ...                           | Regulatory concern | "...SEC inquiry..." |

---

### Risk Narrative

[2–3 paragraphs covering key themes, risk patterns, and notable communications]

---

### Recommended Next Steps

- [ ] Review item #N in full for [specific reason]
- [ ] Escalate to legal counsel regarding [topic]
- [ ] Run `/arctera-audit-pii` if any items may contain PII before export
- [ ] Apply compliance tags to flagged items
```

---

## Behaviour Rules

- Call all applicable search tools **in parallel**, not sequentially.
- Only call `get_email` / `get_file` / `get_message` using IDs from the **current session**.
- Do not fabricate content — only summarise what is in returned snippets or full previews.
- If zero results are found, suggest refined queries and ask the user if they want to retry.
- Do not call get tools for more than 10 items without checking with the user first.
- For `get_file` results, `html_content` is a rendered HTML document — summarise its content, do not display raw HTML.
- For `get_message` results, present the conversation as a chat timeline ordered by `date` ascending.
