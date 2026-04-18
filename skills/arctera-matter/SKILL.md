---
name: arctera-matter
description: Scoped legal matter review — searches all three Arctera archives (email, files, collaboration) filtered by matter name, custodians, keywords, and date range, then produces a structured risk report.
type: skill
---

# Skill: Arctera Legal Matter Review

## Purpose
Run a full scoped review for a named legal matter. Accepts matter name, custodian parties, topic keywords, and a date range. Searches all three archives in parallel, triages the most relevant items, identifies legal and regulatory risks, and delivers a structured matter report with a custodian activity summary and recommended next steps.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Matter Scope

Extract from the user's input:

| Parameter | Identification | Example |
|---|---|---|
| Matter name | Primary noun phrase or after "matter:" | "SEC Investigation", "Project Falcon Litigation" |
| Custodians | Email addresses or after "parties:" / "custodians:" | `john.doe@enron.com`, `jane.smith@enron.com` |
| Date range | Year references or "from … to" | `20010101 TO 20021231` |
| Additional keywords | Any supplementary topic terms | "mark-to-market", "off-balance-sheet" |

If the matter name cannot be determined, stop and ask:
> *"What is the name or description of the legal matter you want to review?"*

---

## Step 2 — Build Lucene Queries

Construct two query variants:

**Full-text query:**
```
ENTIREMESSAGE:"<matter name>" [AND (FROMORTO:"party1@domain.com" OR FROMORTO:"party2@domain.com")] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]] [AND ENTIREMESSAGE:"<keyword>"]
```

**Subject-targeted query:**
```
SUBJECT:"<matter name>" [AND FROMORTO:"<party>"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID
- For domain searches: use `SENDER:enron*` — NEVER `SENDER:*@enron.com`

---

## Step 3 — Execute Searches

Call `search_email`, `search_file`, and `search_message` **in parallel** with both query variants:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate (increment `pageNum` while `has_more` is true) up to 100 total results per archive. If cap is reached, notify the user and offer to narrow the scope.

---

## Step 4 — Triage and Prioritise

From all combined results, select the **top 10 most relevant** items based on:

1. Snippet contains both the matter name and a custodian name
2. Items with `isLegalHold: true`
3. Items with attachments or high `ATTCOUNT`
4. Items involving executives, external counsel, or regulators
5. Items with `CLASSIFICATION.TAGS` already applied

For each selected item, call the appropriate preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

Do not call more than 10 get tools without checking with the user.

---

## Step 5 — Identify Legal Risks

For each previewed item, check for:

| Risk Category | Signals |
|---|---|
| Confidentiality breach | "confidential", "NDA", "do not share", "proprietary", "under seal" |
| Agreement terms | "agreement", "clause", "obligation", "breach", "penalty", "termination" |
| Financial exposure | "settlement", "liability", "damages", "payment", "bonus", "compensation" |
| Regulatory concern | "SEC", "FINRA", "audit", "compliance", "regulatory", "investigation", "subpoena" |
| Communication risk | Hidden/BCC'd parties, external counsel cc'd, unusual sender-recipient routing |
| Legal hold relevance | `isLegalHold: true`, `CLASSIFICATION.TAGS:flagged` or similar |

---

## Step 6 — Produce the Matter Review Report

```
## Arctera Matter Review — [Matter Name]

**Matter:** [name]
**Parties / Custodians:** [list]
**Date range:** [range or "not specified"]
**Search scope:** Emails, Files, Collaboration
**Queries run:** [N]
**Total found:** [N] ([n] emails, [n] files, [n] collab)
**Reviewed:** [N] | **Previewed in full:** [N] | **High-risk:** [N]

---

### High-Risk Items

| # | Archive | Date | Sender / Author | Subject / Filename / Chat Room | Risk Category | Key Excerpt |
|---|---------|------|-----------------|-------------------------------|---------------|-------------|
| 1 | Email | ... | ... | ... | Regulatory concern | "...SEC inquiry..." |

---

### Custodian Activity Summary

| Custodian | Emails | Files | Collab | First Activity | Last Activity | Notable Flags |
|-----------|--------|-------|--------|----------------|---------------|---------------|

---

### Matter Narrative

[2–3 paragraphs covering: key themes, timeline of events, notable communications,
risk patterns, and how the matter evolved across archives]

---

### Recommended Next Steps

- [ ] Review item #N in full for [specific reason]
- [ ] Escalate to legal counsel regarding [topic]
- [ ] Run `/arctera-audit-pii` before any export from this matter
- [ ] Run `/arctera-legal-hold` to identify and confirm hold candidates
- [ ] Run `/arctera-custodian <name>` for a full pull on high-activity custodians
- [ ] Apply compliance tags to all flagged items
```

---

## Behaviour Rules

- Search all three archives **in parallel** — never sequentially.
- Only use `get_email` / `get_file` / `get_message` with IDs from the **current session**.
- Do not fabricate content — only summarise returned snippets or full previews.
- For `get_file` results, `html_content` is rendered HTML — summarise it, do not display raw HTML.
- For `get_message` results, present as a chat timeline ordered by `date` ascending.
- If zero results are found, suggest refined queries and ask the user if they want to retry.
- Do not call more than 10 get tools without checking with the user.
