You are running the **Arctera Communication Timeline** workflow.

User request: $ARGUMENTS

---

## 1 — Parse timeline parameters from $ARGUMENTS

Extract:
- **Topic / keywords** — the subject of the timeline (e.g. "Project Raptor", "mark-to-market accounting")
- **Parties** — optional people to include (e.g. "between ken.lay@enron.com and jeff.skilling@enron.com")
- **Date range** — required or strongly encouraged; prompt if absent (e.g. "2001-01-01 to 2002-01-01")
- **Archive scope** — if the user specifies "emails only", "files only", etc., honour it; otherwise search all three

If no topic or keywords can be identified, stop and ask:
> *"What topic or event would you like to build a timeline for?"*

If no date range is provided, warn the user:
> *"No date range was specified — the timeline may contain a very large number of results. Do you want to proceed or add a date range?"*

---

## 2 — Build Lucene queries

Primary query:
```
ENTIREMESSAGE:"<topic>" [AND FROMORTO:"<party>"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

If multiple keywords, use OR within ENTIREMESSAGE:
```
(ENTIREMESSAGE:"keyword1" OR ENTIREMESSAGE:"keyword2") [AND FROMORTO:...] [AND MAILDATE:...]
```

Sort all results **ascending by date** to build the timeline chronologically.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all applicable archives in parallel

Call `search_email`, `search_file`, and `search_message` simultaneously with:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortField: "date"` and `sortAsc: true` (ascending — oldest first)
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 total results per archive.

---

## 4 — Merge and sort all results chronologically

Combine all results from all archives into a single list sorted by date ascending.
Group by time periods if the range is long (e.g. by month or quarter).

For pivotal items (first occurrence of a key term, items involving senior parties, items with attachments), call the preview tool to retrieve full content:
- Email → `get_email(mailID)`
- File → `get_file(fileId, batchId, messageId)`
- Collab → `get_message(chatRoomId, collabMessageId, chatRoomName, matterId, collabDate)`

---

## 5 — Deliver the Timeline Report

```
## Arctera Communication Timeline — [Topic]

**Topic:** [keywords]
**Parties:** [list or "all parties"]
**Date range:** [range]
**Archives searched:** [Email / Files / Collab / All]
**Total events:** [N] ([n] emails, [n] files, [n] collab)

---

### Timeline

| Date | Archive | Sender / Author | Subject / Filename / Chat Room | Key Excerpt | Flag |
|------|---------|-----------------|-------------------------------|-------------|------|
| [date] | Email | ... | ... | "..." | |
| [date] | File   | ... | ... | "..." | |
| [date] | Collab | ... | ... | "..." | ⚑ |

*(ordered oldest → newest)*

---

### Key Inflection Points

[Narrative of 2–3 paragraphs identifying the most significant moments in the timeline —
first mention, escalation points, change in tone, external parties entering, last activity]

---

### Recommended Next Steps

- [ ] Deep-dive item #N (date: ...) — first occurrence of [keyword]
- [ ] Run `/arctera-thread` for the most active sub-thread
- [ ] Run `/arctera-between` for the most frequent party pair
- [ ] Run `/arctera-matter` if this topic is part of a named legal matter
```

Do not fabricate content — only summarise what is in returned snippets or full previews.
Do not call more than 10 preview tools without checking with the user.
