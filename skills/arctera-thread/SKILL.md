---
name: arctera-thread
description: Reconstructs a full cross-archive conversation thread by correlating email chains, related file shares, and collaboration discussions on the same topic into one unified chronological view.
type: skill
---

# Skill: Arctera Cross-Archive Thread Reconstruction

## Purpose
Reconstruct a complete conversation thread that spans multiple archive types. A topic discussed via email often generates related file shares and parallel chat discussions — this skill finds all three and merges them into a unified view. Identifies cross-archive gaps (e.g. topic discussed in chat but no follow-up email), party overlap across channels, and the full narrative arc of a conversation. Used in eDiscovery, internal investigations, and regulatory response.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Identify the Thread

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Topic / subject | The conversation to reconstruct | Required |
| Key parties | Email addresses or names | Strongly recommended |
| Date range | Narrow window preferred for accuracy | Strongly recommended |
| Anchor item | Specific subject line, filename, or date | Optional |

If no topic or parties can be identified, stop and ask:
> *"What topic or conversation would you like to reconstruct? Providing key parties or a subject line will significantly improve accuracy."*

---

## Step 2 — Phase 1: Locate the Email Thread Anchor

Search email first to find the originating thread:

```
(SUBJECT:"<topic>" OR ENTIREMESSAGE:"<topic>") AND FROMORTO:"<party>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

From results, identify the **thread root**: the earliest matching email. Note:
- `mailID` of the root
- Exact subject line
- All sender and recipient addresses
- Date

---

## Step 3 — Phase 2: Find Related Files

Search for documents shared in the context of this thread, using the topic and the exact date window:

```
ENTIREMESSAGE:"<topic>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Also search for specific attachment names found in the email thread:
```
ATTACHMENTS.FILENAME:"<filename keyword>"
```

---

## Step 4 — Phase 3: Find Related Collaboration Discussions

Search for chat and collaboration messages on the same topic and parties:

```
(ENTIREMESSAGE:"<topic>" OR FROMORTO:"<party>") [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

---

## Step 5 — Execute All Three Searches in Parallel

Call `search_email`, `search_file`, and `search_message` **simultaneously**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — to reconstruct the conversation in order)
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 6 — Preview and Correlate

1. For up to 10 of the most relevant items across all archives, call the appropriate preview tool using **IDs from this session only**:
   - Email → `get_email(mailID: <mailID>)`
   - File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
   - Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

2. Cross-reference across archives:
   - Which parties appear in all three archives?
   - Which topics recur verbatim across email and chat?
   - Are there file versions shared at different points in the thread?

3. Identify **gaps**:
   - Topic active in chat but no corresponding email found within 48 hours
   - Document shared with no accompanying email or chat discussion
   - Party present in email thread who never appears in chat

---

## Step 7 — Produce the Thread Reconstruction Report

```
## Arctera Thread Reconstruction — [Topic]

**Topic:** [subject]
**Key parties:** [list]
**Date range:** [range]
**Thread anchor:** [subject line and date of the earliest email]

**Total items found:** [N] ([n] emails, [n] files, [n] collab)
**Previewed in full:** [N]

---

### Unified Thread Timeline

| # | Date | Archive | From / Author | To / Recipients | Subject / Filename / Room | Key Excerpt |
|---|------|---------|---------------|-----------------|--------------------------|-------------|
*(ordered oldest → newest across all archives)*

---

### Cross-Archive Correlation

| Archive | Items Found | Parties in Common | Date Span |
|---------|-------------|-------------------|-----------|
| Email | N | [names] | [range] |
| Files | N | [names] | [range] |
| Collab | N | [names] | [range] |

**Parties appearing in all three archives:** [names or "none"]

---

### Thread Gaps

- [Date range]: Topic discussed in chat but no corresponding email found within 48 hours
- [Date]: Document shared ([filename]) but no email thread referencing it found
- [Note any other structural gaps]

---

### Thread Narrative

[2–3 paragraphs: how the conversation originated, how it evolved across channels,
key decisions or commitments made, notable tone shifts, and how the thread concluded]

---

### Recommended Next Steps

- [ ] Full preview of item #N — pivotal point (date: ...) — [reason]
- [ ] Investigate thread gap on [date] — no email follow-up to chat discussion
- [ ] Run `/arctera-timeline` for a broader view of this topic
- [ ] Run `/arctera-matter` if this thread is part of a named legal matter
- [ ] Escalate party [name] appearing in all three archives to legal counsel
```

---

## Behaviour Rules

- Run all three archive searches **in parallel**.
- Always sort results ascending by date for correct thread ordering.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- For `get_message` results, present each message as a line in the timeline ordered by `date` ascending.
- If a gap is found (topic in one archive but not another within a narrow time window), always flag it explicitly.
- Do not call more than 10 get tools without checking with the user.
