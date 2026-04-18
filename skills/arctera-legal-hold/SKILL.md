---
name: arctera-legal-hold
description: Identifies legal hold candidates across all three Arctera archives. Flags items already on hold and surfaces new candidates matching matter criteria (custodians, keywords, date range), producing a prioritised hold recommendation report.
type: skill
---

# Skill: Arctera Legal Hold Identification

## Purpose
Identify items in the compliance archive that should be placed on legal hold for a named matter. The skill performs two complementary searches: (1) items already on hold (`isLegalHold: true` or `CLASSIFICATION.TAGS:hold`) and (2) items matching the matter's custodians, keywords, and date range that are not yet held. Delivers a prioritised hold candidate report with per-custodian coverage gaps and specific hold action items.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Hold Parameters

Extract from the user's input:

| Parameter | How to identify | Required? |
|---|---|---|
| Matter name | Primary noun phrase or after "matter:" | Required |
| Custodians | Email addresses or names | Strongly recommended |
| Keywords | Topics in scope for the hold | Recommended |
| Date range | "from … to", year references | Recommended |

If no matter name or keywords can be identified, stop and ask:
> *"Please provide the legal matter name or the topics in scope for this hold."*

---

## Step 2 — Build Queries

Run three distinct query types:

**Query A — Already on hold (system-flagged):**
```
CLASSIFICATION.TAGS:hold [AND (FROMORTO:"custodian1@domain.com" OR FROMORTO:"custodian2@domain.com")] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query B — Candidates by matter name:**
```
ENTIREMESSAGE:"<matter name>" [AND (FROMORTO:"custodian@domain.com")] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query C — Candidates by topic keywords:**
```
(ENTIREMESSAGE:"<keyword1>" OR ENTIREMESSAGE:"<keyword2>") [AND FROMORTO:"<custodian>"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Run Queries A, B, and C across all three archives **in parallel**.

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches

Call `search_email`, `search_file`, and `search_message` for each query in parallel:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 total results per query per archive.

---

## Step 4 — Classify Each Result

For every item returned, assign a Hold Status and Priority:

**Hold Status:**
| Status | Criteria |
|---|---|
| Already held | `isLegalHold: true` OR `CLASSIFICATION.TAGS:hold` in result |
| Hold candidate — High | Custodian match + keyword match + date within range |
| Hold candidate — Medium | Partial match: keyword matches but custodian or date outside scope |
| Not in scope | Keyword only, no custodian or date overlap |

**For all High-priority candidates**, call the appropriate preview tool to confirm relevance before recommending hold placement — using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Legal Hold Report

```
## Arctera Legal Hold Report — [Matter Name]

**Matter:** [name]
**Custodians in scope:** [list]
**Keywords in scope:** [list]
**Date range:** [range]
**Archives searched:** Email, Files, Collaboration

**Items already on legal hold:** [N]
**New hold candidates — High priority:** [N]
**New hold candidates — Medium priority:** [N]
**Items out of scope:** [N]

---

### Items Already on Legal Hold

| # | Archive | Date | Custodian | Subject / Filename / Chat Room | Hold Tag / Status |
|---|---------|------|-----------|-------------------------------|-------------------|

---

### New Hold Candidates — High Priority (recommend immediate hold)

| # | Archive | Date | Custodian | Subject / Filename / Chat Room | Reason | Key Excerpt |
|---|---------|------|-----------|-------------------------------|--------|-------------|

---

### New Hold Candidates — Medium Priority (recommend human review)

| # | Archive | Date | Custodian | Subject / Filename / Chat Room | Reason | Excerpt |
|---|---------|------|-----------|-------------------------------|--------|---------|

---

### Hold Coverage by Custodian

| Custodian | Already Held | New Candidates (High) | New Candidates (Med) | Gap? |
|-----------|-------------|----------------------|----------------------|------|

---

### Recommended Actions

- [ ] Place item #N (Email, mailID: ...) on legal hold — High priority — [reason]
- [ ] Verify hold coverage for custodian [name] — [N] relevant items not yet on hold
- [ ] Review medium-priority candidates with legal counsel before placing on hold
- [ ] Re-run `/arctera-legal-hold` after hold placement to confirm full coverage
- [ ] Run `/arctera-custodian <name>` for a complete pull on each custodian with gaps
```

---

## Behaviour Rules

- Run all three query types across all three archives **in parallel**.
- Always call the preview tool to confirm High-priority candidates before including in the hold recommendation.
- Only use get tools with IDs from the **current session**.
- Do not fabricate hold status — only report `isLegalHold: true` when returned by the archive.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- Do not call more than 10 get tools without checking with the user.
- If zero candidates are found, state: *"No hold candidates found matching the specified criteria in [archives]."*
