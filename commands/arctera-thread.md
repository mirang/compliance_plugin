You are running the **Arctera Cross-Archive Thread Reconstruction** workflow.

User request: $ARGUMENTS

---

## 1 — Identify the thread from $ARGUMENTS

Extract:
- **Topic or subject** — the conversation thread to reconstruct (e.g. "mark-to-market valuation", "Raptor SPE deal")
- **Key parties** — primary participants (e.g. "andrew.fastow@enron.com and jeff.skilling@enron.com")
- **Date range** — narrow window recommended for accurate thread reconstruction
- **Anchor item** — optional: a specific email subject, filename, or date that anchors the thread

If no topic or parties can be identified, stop and ask:
> *"What topic or conversation would you like to reconstruct? Providing key parties or a subject line will improve accuracy."*

---

## 2 — Phase 1: Locate the anchor — search email

Start with `search_email` to find the originating email thread:

```
(SUBJECT:"<topic>" OR ENTIREMESSAGE:"<topic>") AND FROMORTO:"<party>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Retrieve snippets (`contentDetails: 1`, `pageSize: 20`).
Identify the thread root: the earliest matching email. Note its `mailID`, subject, date, and all parties.

---

## 3 — Phase 2: Find related files shared in the thread

Search for documents shared around the same time and topic:

```
ENTIREMESSAGE:"<topic>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Also search for attachment filenames found in the email thread:
```
ATTACHMENTS.FILENAME:"<filename keyword>"
```

---

## 4 — Phase 3: Find related collaboration discussions

Search for chat or collaboration messages on the same topic and parties:

```
(ENTIREMESSAGE:"<topic>" OR FROMORTO:"<party>") [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

---

## 5 — Preview and correlate

Run all three archive searches **in parallel**, then:

1. Preview up to 10 of the most relevant items across all archives using the appropriate get tool
2. Cross-reference: which parties appear in all three archives? Which topics recur?
3. Identify gaps: was something discussed in email but no related file was shared? Was a chat conversation never followed up via email?

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.
Only use IDs obtained in this session.

---

## 6 — Deliver the Thread Reconstruction Report

```
## Arctera Thread Reconstruction — [Topic]

**Topic:** [subject]
**Key parties:** [list]
**Date range:** [range]
**Thread anchor:** [earliest email subject and date]

---

### Unified Thread Timeline

| Date | Archive | From / Author | To / Recipients | Subject / Filename / Room | Key Excerpt |
|------|---------|---------------|-----------------|--------------------------|-------------|
*(ordered oldest → newest across all archives)*

---

### Cross-Archive Correlation

| Archive | Items Found | Parties Overlap | Date Range |
|---------|-------------|-----------------|------------|
| Email   | N           | [names]         | [range]    |
| Files   | N           | [names]         | [range]    |
| Collab  | N           | [names]         | [range]    |

---

### Thread Gaps

[Bullet list of any gaps found — e.g. "Topic discussed in chat on [date] but no corresponding email found within 48 hours"]

---

### Thread Narrative

[2–3 paragraphs: how the conversation evolved, key decisions made, tone shifts, external parties]

---

### Recommended Next Steps

- [ ] Preview full content of item #N — pivotal point in thread
- [ ] Run `/arctera-timeline` for a broader date range on this topic
- [ ] Run `/arctera-matter` if this thread is part of a named legal matter
- [ ] Escalate thread gap on [date] to legal counsel
```
