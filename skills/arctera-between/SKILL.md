---
name: arctera-between
description: Finds all communications between two specific people across all three Arctera archives. Produces a relationship communication report covering volume, timeline, topics, third parties, and notable items — core to investigating undisclosed relationships and side-channel communications.
type: skill
---

# Skill: Arctera Point-to-Point Communication Audit

## Purpose
Retrieve all communications between two specific individuals across email, files, and collaboration. Builds a complete picture of the relationship: frequency, date range, topics discussed, third parties included, and any items on legal hold. Used in eDiscovery to investigate undisclosed relationships, side-channel communications, conflicts of interest, and insider risk — as well as in general custodian interview preparation.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Identify Both Parties

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Party A | Email address or full name | Required |
| Party B | Email address or full name | Required |
| Date range | "from … to", year references | Optional |
| Topic filter | Additional keyword | Optional |
| Archive scope | "emails", "all", etc. | Default: all three |

If both parties cannot be identified, stop and ask:
> *"Please provide the email addresses or names of both parties whose communications you want to review."*

---

## Step 2 — Build Bidirectional Queries

### Email — capture both directions:
```
(SENDER:"partyA@domain.com" AND FROMORTO:"partyB@domain.com") OR (SENDER:"partyB@domain.com" AND FROMORTO:"partyA@domain.com") [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]] [AND ENTIREMESSAGE:"keyword"]
```

### Files — both names present in the document:
```
ENTIREMESSAGE:"Party A Name" AND ENTIREMESSAGE:"Party B Name" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

### Collaboration — sender in either direction, name in content:
```
(FROMORTO:"partyA@domain.com" AND ENTIREMESSAGE:"Party B Name") OR (FROMORTO:"partyB@domain.com" AND ENTIREMESSAGE:"Party A Name") [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
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
- `sortAsc: true` (ascending — to build a relationship timeline)
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per archive.

---

## Step 4 — Triage and Prioritise

From all results, identify:

1. **Frequency pattern**: are communications regular or clustered around specific events?
2. **Topic distribution**: what subjects recur across their communications?
3. **Third parties included**: who else appears in their communications? Any unexpected names?
4. **External leakage**: any items sent externally by either party to non-corporate domains?
5. **Legal hold items**: any items flagged `isLegalHold: true`?
6. **Communication pattern anomalies**: sudden spike in activity, shift to chat/collaboration, "delete this" language

Select the **top 10 most significant items** for full preview using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Point-to-Point Relationship Report

```
## Arctera Communication Audit — [Party A] ↔ [Party B]

**Party A:** [Full name and email]
**Party B:** [Full name and email]
**Date range searched:** [range or "all available dates"]
**Topic filter:** [keyword or "none — full pull"]
**Archives searched:** Email, Files, Collaboration

**Total communications found:** [N] ([n] emails, [n] files, [n] collab)
**First contact:** [date and archive]
**Last contact:** [date and archive]
**Items on legal hold:** [N]

---

### Communication Summary

| Archive | Count | A → B | B → A | Date Range |
|---------|-------|-------|-------|------------|
| Email | N | N | N | range |
| Files | N | — | — | range |
| Collab | N | N | N | range |

---

### Communication Timeline

| # | Date | Archive | From | To | Subject / Room | Key Excerpt | Flag |
|---|------|---------|------|----|----------------|-------------|------|
*(ordered oldest → newest)*

---

### Topic Distribution

| Topic / Theme | Frequency | Archive(s) |
|---|---|---|

---

### Third Parties Included

| Third Party | Archive | Count | Context |
|-------------|---------|-------|---------|

---

### Relationship Narrative

[2–3 paragraphs: nature and frequency of the relationship, key topics discussed,
whether communications show a consistent pattern or anomalies, any notable third-party involvement]

---

### Recommended Next Steps

- [ ] Full review of item #N — [reason it is pivotal]
- [ ] Run `/arctera-timeline` to map this relationship against corporate events
- [ ] Run `/arctera-matter <matter name>` if this relationship is relevant to a legal matter
- [ ] Investigate third party [name] — appears in [N] communications between the two parties
- [ ] Run `/arctera-channel-audit` if unusual volume was found in collaboration channels
```

---

## Behaviour Rules

- Build **bidirectional** queries — capture both A→B and B→A directions.
- Run all three archive searches **in parallel**.
- Sort results ascending by date — the relationship timeline is the primary output.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- For `get_message` results, present as a chat timeline ordered by `date` ascending.
- Do not call more than 10 get tools without checking with the user.
