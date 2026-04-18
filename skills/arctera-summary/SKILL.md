---
name: arctera-summary
description: Generates an executive summary report across a matter or topic for non-legal stakeholders. Searches all three archives, triages the most significant items, and produces a plain-English narrative with key findings, risk indicators, and recommended actions — written for board, audit committee, or senior management audiences.
type: skill
---

# Skill: Arctera Executive Summary Report

## Purpose
Produce a clear, concise executive summary of archived communications on a topic or matter — written for a non-legal, senior management audience. No raw PII, no Lucene syntax, no archive jargon in the output. Searches all three archives, identifies the most significant findings, and delivers a board-ready narrative with key findings, risk level, and recommended actions. Typically used as the entry point before a full legal review, or to brief senior stakeholders on investigation findings.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Summary Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Matter / topic | Primary subject | Required |
| Date range | Year or date range | Optional |
| Audience | "board", "audit committee", "regulators", "management" | Optional — adjust tone accordingly |
| Archive scope | Default: all three | Optional |

If no topic can be identified, stop and ask:
> *"What matter or topic would you like an executive summary for?"*

---

## Step 2 — Build the Summary Query

```
ENTIREMESSAGE:"<topic>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Also run a subject-targeted variant:
```
SUBJECT:"<topic>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call `search_email`, `search_file`, and `search_message` **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to **60 total results per archive** — sufficient for an executive summary (not a full review).

---

## Step 4 — Identify the 5 Most Significant Items

From all results, select the **5 most significant** items based on:

1. Explicit risk language (regulatory, legal, financial, confidentiality)
2. Senior executive or external counsel involvement
3. `isLegalHold: true`
4. Earliest and most recent items (to bound the timeline)
5. Items with attachments indicating formal documentation

For each selected item, call the appropriate preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Executive Summary Report

Write for a **non-legal senior management audience**:
- No Lucene queries in the output
- No raw PII values anywhere
- No archive IDs visible to the reader
- No compliance jargon — plain English throughout
- Adjust formality for the specified audience

```
## Executive Summary — [Topic / Matter]

**Prepared for:** [audience or "Senior Management"]
**Prepared by:** Arctera Compliance Archive — AI-assisted review
**Date:** [today's date — YYYY-MM-DD]
**Period covered:** [date range]
**Archives reviewed:** Email, Files, Collaboration

---

### Overview

[1 paragraph: what was reviewed, why, and the headline conclusion.
E.g.: "This summary covers archived communications related to [topic] from [date range].
The review identified [N] relevant communications across email, documents, and collaboration channels.
The overall risk level is assessed as [High / Medium / Low] based on the findings below."]

---

### Key Findings

[5 bullet points maximum — one sentence each, plain English, no jargon]

- [Finding 1 — most significant]
- [Finding 2]
- [Finding 3]
- [Finding 4]
- [Finding 5]

---

### Volume of Communications

| Channel | Records Found | Date Range | Key Individuals |
|---------|---------------|------------|-----------------|
| Email | N | range | [first names or roles — no raw email addresses] |
| Documents | N | range | [first names or roles] |
| Chat / Messaging | N | range | [first names or roles] |
| **Total** | **N** | | |

---

### Risk Assessment

| Risk Area | Level | Summary |
|---|---|---|
| Legal / Regulatory | 🔴 High / 🟡 Medium / 🟢 Low | [one sentence plain English] |
| Data Privacy | 🔴 High / 🟡 Medium / 🟢 Low | [one sentence] |
| Financial Exposure | 🔴 High / 🟡 Medium / 🟢 Low | [one sentence] |
| Communication Risk | 🔴 High / 🟡 Medium / 🟢 Low | [one sentence] |

---

### Narrative

[2–3 paragraphs. Plain English. Cover:
- What the communications show about the topic
- Who was involved and what decisions or actions were taken
- What the key risks or concerns are and why they matter
No technical compliance language. Write as if briefing a CEO or board member.]

---

### Recommended Actions

[3–5 clear, actionable recommendations. Who does what, and with what urgency.]

1. **[Immediate]** [Action — what, who, why]
2. **[Short-term]** [Action]
3. **[Ongoing]** [Action]

---

### Notes for the Legal and Compliance Team

*This summary is based on an automated first-pass review and is intended for executive briefing purposes only. It does not constitute legal advice and should not be relied upon as a complete review.*

- For full legal review: `/arctera-matter [topic]`
- For PII audit before export: `/arctera-audit-pii`
- For legal hold assessment: `/arctera-legal-hold`
- For regulatory red flag scan: `/arctera-regulatory`
```

---

## Behaviour Rules

- Search all three archives **in parallel**.
- Only preview the **5 most significant** items — this is a summary, not a full review.
- Only use get tools with IDs from the **current session**.
- **Never include raw PII values** in the report — describe findings at a category level.
- **Never include Lucene queries** in the report output — the audience is non-technical.
- **Never include archive IDs** in the main body — they may appear only in the "Notes for Legal Team" section.
- For `get_file` results, summarise `html_content` in plain language — do not display raw HTML.
- Adjust formality for audience: "board" → more formal; "management" → clear and direct.
- Always include the disclaimer in the Notes section.
