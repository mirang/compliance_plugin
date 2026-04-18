You are running the **Arctera Cross-Archive Topic Correlation** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Topic / keywords** — the subject to correlate across archives
- **Date range** — optional scope
- **Parties** — optional custodians to filter by

If no topic can be identified, stop and ask:
> *"What topic or keywords would you like to correlate across all three archives?"*

---

## 2 — Build the unified query

Use the same query for all three archives:
```
ENTIREMESSAGE:"<topic>" [AND (FROMORTO:"party@domain.com")] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Also build a subject-targeted query for email:
```
SUBJECT:"<topic>" [AND FROMORTO:"<party>"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all three archives in parallel

Call `search_email`, `search_file`, and `search_message` simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — to show chronological evolution)
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 results per archive.

---

## 4 — Analyse and correlate

From the combined results:
1. **Presence**: which archives contain the topic? Email only? All three? Files only?
2. **Timeline**: when did the topic appear in each archive? Is there a progression (email → file → chat)?
3. **Parties**: who appears in each archive discussing this topic? Any parties unique to one archive?
4. **Volume**: where is the most activity concentrated?
5. **Gaps**: is the topic heavily discussed in one archive but absent from another?

For the top 5 most significant items per archive, call the preview tool:
- Email → `get_email(mailID)`
- File → `get_file(fileId, batchId, messageId)`
- Collab → `get_message(chatRoomId, collabMessageId, chatRoomName, matterId, collabDate)`

Only use IDs from this session.

---

## 5 — Deliver the Cross-Archive Correlation Report

```
## Arctera Cross-Archive Correlation — [Topic]

**Topic:** [keywords]
**Date range:** [range or "all dates"]
**Archives searched:** Email, Files, Collaboration
**Total found:** [N] ([n] emails, [n] files, [n] collab)

---

### Archive Presence Summary

| Archive | Count | Date Range | Top Parties | Key Observations |
|---------|-------|------------|-------------|-----------------|

---

### Chronological Evolution

[How the topic progressed across archives over time — e.g. "First discussed via email in [month],
a related document was shared in [month], chat discussions began in [month]"]

---

### Parties Analysis

| Party | Email | Files | Collab | Total | Note |
|-------|-------|-------|--------|-------|------|

---

### Archive Gaps

- [Topic present in Email and Files but absent from Collaboration — note and flag]
- [Topic in Collab but no supporting file found — note and flag]

---

### Top Items per Archive

*(3–5 most relevant items from each archive with excerpt)*

---

### Recommended Next Steps

- [ ] Run `/arctera-thread` to reconstruct the full conversation starting from [earliest email]
- [ ] Run `/arctera-timeline` for a chronological view of all events
- [ ] Run `/arctera-matter` if this topic is part of a named legal matter
- [ ] Investigate gap: [archive] has [N] items but [other archive] has 0 — may indicate side-channel use
```
