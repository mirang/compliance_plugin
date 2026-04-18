You are running the **Arctera Legal Matter Review** workflow.

User request: $ARGUMENTS

---

## 1 — Parse matter scope from $ARGUMENTS

Extract the following from $ARGUMENTS:

| Parameter | How to identify | Example |
|---|---|---|
| **Matter name** | Text after "matter:", or the primary noun phrase | "SEC Investigation", "Project Falcon Litigation" |
| **Parties / Custodians** | Text after "parties:", "custodians:", or email addresses | "john.doe@enron.com, jane.smith@enron.com" |
| **Date range** | Text after "date:", "from … to", or year references | "2001-01-01 to 2002-06-30" |
| **Keywords** | Any additional topic terms | "mark-to-market", "SPE", "off-balance-sheet" |

If **matter name** cannot be identified, stop and ask:
> *"What is the name or description of the legal matter you want to review?"*

---

## 2 — Build Lucene queries

Construct the base query using all extracted parameters:

```
ENTIREMESSAGE:"<matter name>" [AND FROMORTO:"<party-email>"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]] [AND ENTIREMESSAGE:"<keyword>"]
```

Also build a subject-targeted variant:
```
SUBJECT:"<matter name>" [AND FROMORTO:"<party-email>"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Run both queries. If multiple parties are provided, run one query per party OR combine with OR:
```
FROMORTO:"party1@domain.com" OR FROMORTO:"party2@domain.com"
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all three archives in parallel

Call `search_email`, `search_file`, and `search_message` simultaneously with:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 total results per archive.

---

## 4 — Triage and preview top items

From combined results, select the top 10 most relevant items based on:
1. Snippet contains the matter name and multiple party names
2. Items marked `isLegalHold: true`
3. Items with attachments or high `ATTCOUNT`
4. Items involving executives or external counsel

Call the appropriate preview tool for each:
- Email → `get_email(mailID)`
- File → `get_file(fileId, batchId, messageId)`
- Collab → `get_message(chatRoomId, collabMessageId, chatRoomName, matterId, collabDate)`

Only use IDs obtained in this session.

---

## 5 — Identify risks per item

For each previewed item, flag:
- **Confidentiality**: "confidential", "NDA", "do not share", "proprietary"
- **Agreement terms**: "agreement", "obligation", "breach", "penalty", "termination"
- **Financial exposure**: "settlement", "liability", "damages", "payment"
- **Regulatory**: "SEC", "FINRA", "audit", "investigation", "subpoena"
- **Communication risk**: BCC'd parties, external counsel, unusual routing

---

## 6 — Deliver the Matter Review Report

```
## Arctera Matter Review — [Matter Name]

**Matter:** [name]
**Parties / Custodians:** [list]
**Date range:** [range or "not specified"]
**Search scope:** Emails, Files, Collaboration
**Query:** `[Lucene query used]`
**Total found:** [N] ([n] emails, [n] files, [n] collab)
**Reviewed:** [N] | **Previewed in full:** [N] | **High-risk:** [N]

---

### High-Risk Items

| # | Archive | Date | Sender / Author | Subject / Filename / Chat Room | Risk Category | Key Excerpt |
|---|---------|------|-----------------|-------------------------------|---------------|-------------|

---

### Custodian Activity Summary

| Custodian | Emails | Files | Collab | First Seen | Last Seen | Notable |
|-----------|--------|-------|--------|------------|-----------|---------|

---

### Matter Narrative

[2–3 paragraphs: key themes, timeline of events, notable communications, risk patterns]

---

### Recommended Next Steps

- [ ] Review item #N in full for [reason]
- [ ] Escalate to legal counsel regarding [topic]
- [ ] Run `/arctera-audit-pii` before any export
- [ ] Run `/arctera-legal-hold` to identify hold candidates
- [ ] Apply compliance tags to flagged items
```

Do not fabricate content. Summarise `html_content` from `get_file` results — do not display raw HTML.
Present `get_message` results as a chat timeline ordered by date ascending.
