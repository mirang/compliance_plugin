---
name: arctera-cross-archive
description: Correlates a topic across all three Arctera archives simultaneously. Surfaces how a topic evolved from email → files → collaboration, identifies which archives are active, flags cross-archive gaps, and produces a unified correlation report.
type: skill
---

# Skill: Arctera Cross-Archive Topic Correlation

## Purpose
Search a topic across all three archives in parallel and correlate the results into a unified picture. Identifies which channels were used to discuss the topic, how it progressed across archives over time, which parties appear in multiple archives, and where notable gaps exist (e.g. heavy chat activity with no corresponding email). Used for eDiscovery scoping, investigation prioritisation, and channel risk assessment.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Topic Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Topic / keywords | Primary subject | Required |
| Date range | Year or date range | Optional |
| Parties | Email addresses or names | Optional |

If no topic can be identified, stop and ask:
> *"What topic or keywords would you like to correlate across all three archives?"*

---

## Step 2 — Build the Unified Query

Use the **same query** for all three archives:
```
ENTIREMESSAGE:"<topic>" [AND (FROMORTO:"party@domain.com")] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Also run a subject-targeted variant for email:
```
SUBJECT:"<topic>" [AND FROMORTO:"<party>"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

For multi-keyword topics:
```
(ENTIREMESSAGE:"keyword1" OR ENTIREMESSAGE:"keyword2") [AND FROMORTO:...] [AND MAILDATE:...]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute All Searches in Parallel

Call `search_email`, `search_file`, and `search_message` **simultaneously**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — for chronological correlation)
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per archive.

---

## Step 4 — Analyse and Correlate

From the combined results, perform the following analysis:

**1. Presence**: which archives contain the topic?
| Archive | Present? | Count |
|---|---|---|
| Email | Yes/No | N |
| Files | Yes/No | N |
| Collab | Yes/No | N |

**2. Timeline**: when did the topic first appear in each archive? Is there a logical progression (email → document → chat)?

**3. Parties**: who discusses this topic in each archive? Do the same parties appear across all three, or is the topic discussed by different groups in different channels?

**4. Volume distribution**: where is the most discussion concentrated? A topic heavily discussed in chat but not in email may indicate intentional avoidance of email.

**5. Gaps**: flag any archive where the topic is absent or significantly underrepresented relative to the others.

---

## Step 5 — Preview Top Items

For the top 5 most significant items **per archive** (up to 15 total), call the preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

Do not preview more than 15 items total without checking with the user.

---

## Step 6 — Produce the Cross-Archive Correlation Report

```
## Arctera Cross-Archive Correlation — [Topic]

**Topic:** [keywords]
**Date range:** [range or "all dates"]
**Parties filter:** [list or "all parties"]
**Total found:** [N] ([n] emails, [n] files, [n] collab)

---

### Archive Presence Summary

| Archive | Count | Date Range | Top Parties | Activity Level |
|---------|-------|------------|-------------|----------------|
| Email | N | range | names | High / Medium / None |
| Files | N | range | names | High / Medium / None |
| Collab | N | range | names | High / Medium / None |

---

### Chronological Evolution

[Narrative: when did the topic first appear in each archive? How did it move across channels?
E.g. "Topic first appeared in email in [month]. A related spreadsheet was shared in [month].
Chat discussions spiked in [month] with no corresponding email activity."]

---

### Parties Across Archives

| Party | Email | Files | Collab | All Three? | Note |
|-------|-------|-------|--------|------------|------|

---

### Cross-Archive Gaps

| Gap | Description | Risk Implication |
|---|---|---|
| Chat activity without email | [N] collab messages with no corresponding email in same window | May indicate deliberate avoidance of archived channels |
| Document with no discussion | [N] files with no related email or chat thread | May indicate out-of-band communication |

---

### Top Items per Archive

**Email (top 3):**
| # | Date | Sender | Subject | Key Excerpt |
|---|------|--------|---------|-------------|

**Files (top 3):**
| # | Date | Author | Filename | Key Excerpt |
|---|------|--------|----------|-------------|

**Collab (top 3):**
| # | Date | Sender | Chat Room | Key Excerpt |
|---|------|--------|-----------|-------------|

---

### Recommended Next Steps

- [ ] Run `/arctera-thread` to reconstruct the full conversation from [earliest email]
- [ ] Run `/arctera-timeline` for a unified chronological view across all archives
- [ ] Investigate gap: [archive] has [N] items but [other archive] has 0 — possible side-channel use
- [ ] Run `/arctera-matter <topic>` if this is part of a named legal matter
- [ ] Run `/arctera-channel-audit` on the collaboration channel with highest unexplained volume
```

---

## Behaviour Rules

- Always search all three archives **in parallel** using the same query — this is the defining characteristic of this skill.
- Sort results ascending by date for chronological correlation analysis.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- For `get_message` results, present as a chat timeline ordered by `date` ascending.
- Always explicitly note when an archive has zero results — absence is as significant as presence.
