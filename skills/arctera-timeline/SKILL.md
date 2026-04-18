---
name: arctera-timeline
description: Builds a chronological communication timeline around a topic or between parties across all three Arctera archives. Merges email, file, and collaboration results sorted oldest-to-newest to reconstruct the sequence of events.
type: skill
---

# Skill: Arctera Communication Timeline

## Purpose
Reconstruct the chronological sequence of communications around a topic, event, or between specific parties. All results from email, files, and collaboration are merged and ordered oldest-to-newest to create a unified event timeline. Pivotal moments — first mentions, escalations, tone shifts, external party entries — are highlighted with narrative context. Useful for litigation fact-finding, regulatory response, and internal investigations.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Timeline Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Topic / keywords | Primary subject of the timeline | Required |
| Parties | Email addresses or names mentioned | Optional |
| Date range | "from … to", year references | Strongly recommended — prompt if absent |
| Archive scope | "emails only", "all", etc. | Default: all three |

If no topic can be identified, stop and ask:
> *"What topic or event would you like to build a timeline for?"*

If no date range is provided, warn:
> *"No date range was specified — results may be very large. Do you want to proceed or add a date range first?"*

---

## Step 2 — Build Lucene Queries

**Primary query (full-text):**
```
ENTIREMESSAGE:"<topic>" [AND (FROMORTO:"party1@domain.com" OR FROMORTO:"party2@domain.com")] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Subject-targeted query (email only):**
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

## Step 3 — Execute Searches (Ascending Date Order)

Call `search_email`, `search_file`, and `search_message` **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortField: "date"` and `sortAsc: true` ← **critical for timeline ordering**
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 total results per archive.

---

## Step 4 — Merge and Sort

1. Combine all results from all three archives into a single list
2. Sort the merged list by date **ascending** (oldest event first)
3. Assign a sequential event number to each item
4. Group by time period if the range spans more than 6 months (e.g. by month or quarter)

---

## Step 5 — Identify Pivotal Events

From the merged timeline, identify:

| Event Type | How to identify |
|---|---|
| First mention | Earliest item containing the topic keyword |
| Escalation point | First item involving a more senior party or external recipient |
| Tone shift | Language shifts from neutral to urgent, defensive, or legal |
| External party entry | First item to/from a non-corporate domain |
| Last activity | Latest item — when did communications stop? |
| Archive gap | Long silence between events, or topic active in one archive but not others |

For pivotal events, call the preview tool to retrieve full content:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

Only use IDs from the current session. Do not preview more than 10 items without checking with the user.

---

## Step 6 — Produce the Timeline Report

```
## Arctera Communication Timeline — [Topic]

**Topic:** [keywords]
**Parties:** [list or "all parties in results"]
**Date range:** [range or "not specified"]
**Archives searched:** [Email / Files / Collab / All]
**Total events:** [N] ([n] emails, [n] files, [n] collab)
**Pivotal events identified:** [N]

---

### Event Timeline

| # | Date | Archive | Sender / Author | Subject / Filename / Chat Room | Key Excerpt | Flag |
|---|------|---------|-----------------|-------------------------------|-------------|------|
| 1 | date | Email  | ... | ... | "first mention of..." | ★ First |
| 2 | date | Collab | ... | ... | "..." | |
| 3 | date | File   | ... | ... | "..." | ★ External |

*(ordered oldest → newest)*

---

### Key Inflection Points

**First mention:** [Date] — [Archive] — [Brief description]
**Escalation:** [Date] — [Archive] — [Brief description]
**External party entry:** [Date] — [Who] — [Archive]
**Last activity:** [Date] — [Archive] — [Brief description]
**Gaps:** [Any notable silences or archive gaps]

---

### Timeline Narrative

[2–3 paragraphs: how the topic evolved over time, who drove the conversation at each stage,
notable changes in tone or urgency, and what the timeline suggests about decision-making]

---

### Recommended Next Steps

- [ ] Review full content of event #N — [reason it is pivotal]
- [ ] Run `/arctera-thread` to reconstruct the sub-conversation starting at event #N
- [ ] Run `/arctera-between <party A> and <party B>` for the most frequent pair
- [ ] Run `/arctera-matter` if this timeline is part of a named legal matter
- [ ] Flag gap between [date] and [date] for legal counsel review
```

---

## Behaviour Rules

- Always sort results **ascending by date** — this is the defining characteristic of a timeline.
- Search all applicable archives **in parallel**.
- If only one archive is in scope (user specified), search only that archive.
- Only use `get_email` / `get_file` / `get_message` with IDs from the **current session**.
- Do not fabricate events — only report what is in the returned data.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- For `get_message` results, present as a chat timeline ordered by `date` ascending.
- If zero results are found in any archive, explicitly note the absence.
