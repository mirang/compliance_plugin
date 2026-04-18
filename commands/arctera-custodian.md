You are running the **Arctera Custodian Communication Pull** workflow.

User request: $ARGUMENTS

---

## 1 — Identify the custodian from $ARGUMENTS

Extract:
- **Email address** — primary identifier (e.g. `john.doe@enron.com`)
- **Full name** — used for file and chat searches (e.g. "John Doe")
- **Date range** — optional (`MAILDATE:[YYYYMMDD TO YYYYMMDD]`)
- **Topic filter** — optional additional keyword to scope the pull

If neither an email address nor a full name can be identified, stop and ask:
> *"Please provide the custodian's email address or full name."*

---

## 2 — Build Lucene queries per archive

**Email** (sent or received):
```
FROMORTO:"custodian@domain.com" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]] [AND ENTIREMESSAGE:"keyword"]
```

**Files** (authored or shared):
```
ENTIREMESSAGE:"First Last" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```
Also try:
```
SENDER:"custodian@domain.com" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Collaboration / Chat**:
```
FROMORTO:"custodian@domain.com" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```
Also try full name:
```
ENTIREMESSAGE:"First Last" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.
For domain-level sender searches use `SENDER:domain*` not `SENDER:*@domain.com`.

---

## 3 — Search all three archives in parallel

Call `search_email`, `search_file`, and `search_message` simultaneously with:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 total results per archive.

---

## 4 — Triage and prioritise

From all results, flag the most significant items:
1. Items with `isLegalHold: true`
2. External parties in the conversation (non-corporate domains)
3. Items involving executives or regulators
4. Items with large attachments or multiple recipients
5. Earliest and latest items per archive (to establish the custodian's communication window)

For the top 10 most significant items, call the appropriate preview tool:
- Email → `get_email(mailID)`
- File → `get_file(fileId, batchId, messageId)`
- Collab → `get_message(chatRoomId, collabMessageId, chatRoomName, matterId, collabDate)`

Only use IDs obtained in this session.

---

## 5 — Deliver the Custodian Profile Report

```
## Arctera Custodian Report — [Custodian Name / Email]

**Custodian:** [name and email]
**Date range searched:** [range or "all dates"]
**Topic filter:** [keyword or "none"]
**Archives searched:** Email, Files, Collaboration

**Total found:** [N] ([n] emails, [n] files, [n] collab)
**Reviewed:** [N] | **Previewed in full:** [N]
**On legal hold:** [N items flagged isLegalHold]
**External communications:** [N items to/from non-corporate domains]

---

### Communication Activity Summary

| Archive | Count | Date Range | Top Counterparties |
|---------|-------|------------|--------------------|

---

### Notable Items

| # | Archive | Date | With / Subject / Filename | Key Excerpt | Flag |
|---|---------|------|--------------------------|-------------|------|

---

### External Contacts

| External Party | Archive | Count | First Contact | Last Contact |
|----------------|---------|-------|---------------|--------------|

---

### Recommended Next Steps

- [ ] Run `/arctera-between <custodian> and <counterparty>` to deep-dive key relationships
- [ ] Run `/arctera-legal-hold` to confirm hold coverage for this custodian
- [ ] Run `/arctera-audit-pii` if this custodian's records are being exported
- [ ] Escalate notable external communications to legal counsel
```

Do not fabricate content. Summarise `html_content` from `get_file` — do not display raw HTML.
Present collab results as a chat timeline ordered by date ascending.
