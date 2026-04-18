---
name: arctera-project
description: Pulls all communications related to a named project, deal, or initiative across all three Arctera archives. Produces a chronological project communication report covering key milestones, participants, external parties, and associated documents.
type: skill
---

# Skill: Arctera Project & Deal Communication Pull

## Purpose
Retrieve and organise all archived communications related to a specific named project, deal, transaction, or corporate initiative. Searches across email, files, and collaboration simultaneously, reconstructs the project's communication history chronologically, and identifies key participants, decision points, external parties, and supporting documents. Used in eDiscovery, M&A due diligence review, regulatory response, and internal project audits.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Identify the Project

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Project / deal name | Named initiative (e.g. "Project Raptor", "LJM Partnership") | Required |
| Code name / abbreviation | Shortened name or internal code | Optional additional query |
| Date range | "from … to", year references | Optional |
| Key custodians | Email addresses or names | Optional |

If no project name can be identified, stop and ask:
> *"What is the name of the project or deal you want to pull communications for?"*

---

## Step 2 — Build Project Queries

Run all variants to maximise recall:

**Full quoted phrase:**
```
ENTIREMESSAGE:"<Project Name>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Subject-targeted (email):**
```
SUBJECT:"<Project Name>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Abbreviation / code name (if identifiable):**
```
ENTIREMESSAGE:"<Code Name>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Custodian-scoped variant (if custodians provided):**
```
ENTIREMESSAGE:"<Project Name>" AND FROMORTO:"custodian@domain.com" [AND MAILDATE:...]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call `search_email`, `search_file`, and `search_message` **in parallel** for each query variant:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — to reconstruct the project chronologically)
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per archive.

---

## Step 4 — Triage and Prioritise

From all results, identify:

1. **Inception communications**: the earliest items — project kick-off, initial approvals
2. **Key decision points**: items referencing approvals, sign-offs, milestones, terms
3. **External parties**: non-corporate domain participants (advisors, counterparties, regulators)
4. **Files and documents**: contracts, models, presentations, term sheets
5. **Legal hold items**: `isLegalHold: true`
6. **Risk signals**: regulatory references, confidentiality concerns, unusual routing

Select the **top 10 most significant items** for full preview using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Project Communication Report

```
## Arctera Project Report — [Project Name]

**Project:** [name and any code name]
**Date range:** [range or "all available dates"]
**Custodians in scope:** [list or "all participants"]
**Archives searched:** Email, Files, Collaboration

**Total communications found:** [N] ([n] emails, [n] files, [n] collab)
**Date span:** [earliest] to [latest]
**Reviewed:** [N] | **Previewed in full:** [N]
**Items on legal hold:** [N]

---

### Project Timeline

| # | Date | Archive | Sender / Author | Subject / Filename / Room | Key Excerpt | Milestone? |
|---|------|---------|-----------------|--------------------------|-------------|------------|
| 1 | date | Email | ... | "Project kick-off..." | "..." | ★ Inception |
| 2 | date | File  | ... | "Term sheet v1.xlsx" | "..." | ★ Document |
| 3 | date | Collab| ... | "LJM discussion" | "..." | |

*(ordered oldest → newest)*

---

### Key Participants

| Party | Archive(s) Active In | Role / Context | Item Count | First Activity | Last Activity |
|-------|---------------------|----------------|------------|----------------|---------------|

---

### External Parties

| External Party / Domain | Archive | Role | First Contact | Last Contact | Item Count |
|------------------------|---------|------|---------------|--------------|------------|

---

### Documents & Files

| # | Date | Author | Filename | Key Content Summary |
|---|------|--------|----------|---------------------|

---

### Project Narrative

[2–3 paragraphs: project origin and structure, key milestones and decision points,
parties driving the project, external relationships, and any risk signals identified]

---

### Recommended Next Steps

- [ ] Run `/arctera-timeline` for the complete chronological event view
- [ ] Run `/arctera-between <key party A> and <key party B>` for critical relationship pairs
- [ ] Run `/arctera-legal-hold` if this project is subject to active litigation or investigation
- [ ] Run `/arctera-audit-pii` before exporting project communications for production
- [ ] Run `/arctera-regulatory` if regulatory references were found in project communications
```

---

## Behaviour Rules

- Always search all three archives **in parallel**.
- Sort results ascending by date — chronological reconstruction is the primary value.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- For `get_message` results, present as a chat timeline ordered by `date` ascending.
- If zero results are found in any archive, note it explicitly — absence is meaningful for a project audit.
- Do not call more than 10 get tools without checking with the user.
