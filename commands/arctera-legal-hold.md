You are running the **Arctera Legal Hold Identification** workflow.

User request: $ARGUMENTS

---

## 1 — Parse hold parameters from $ARGUMENTS

Extract:
- **Matter name** — the legal matter requiring a hold (e.g. "SEC Investigation 2001")
- **Custodians** — specific people whose records must be held (email addresses or names)
- **Keywords** — topics in scope for the hold (e.g. "mark-to-market", "Project Raptor")
- **Date range** — the relevant time period for the hold
- **Archive scope** — default to all three archives unless user specifies otherwise

If no matter name or keywords can be identified, stop and ask:
> *"Please provide the legal matter name or the topics in scope for this hold."*

---

## 2 — Build hold identification queries

**Query 1 — Already on hold** (items flagged in the system):
```
CLASSIFICATION.TAGS:hold [AND FROMORTO:"custodian@domain.com"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 2 — Candidates by matter keyword**:
```
ENTIREMESSAGE:"<matter name>" [AND FROMORTO:"custodian@domain.com"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 3 — Candidates by topic keywords**:
```
(ENTIREMESSAGE:"<keyword1>" OR ENTIREMESSAGE:"<keyword2>") AND FROMORTO:"<custodian>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Run queries 2 and 3 in parallel across all three archives.
Query 1 can run simultaneously.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all archives in parallel

Call `search_email`, `search_file`, and `search_message` with:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 total results.

---

## 4 — Classify each result

For every item returned, assign:

**Hold Status:**
| Status | Criteria |
|---|---|
| Already held | `isLegalHold: true` OR `CLASSIFICATION.TAGS:hold` present |
| Hold candidate | Matches matter keywords and custodian, not yet on hold |
| Not in scope | Keyword match only, no custodian or date overlap |

**Priority:**
| Priority | Criteria |
|---|---|
| High | Custodian match + keyword match + date in range |
| Medium | Partial match — keyword but wrong party, or date outside range |
| Low | Keyword only, no custodian match |

For all **High priority hold candidates**, call the appropriate preview tool to confirm relevance before recommending hold placement.

---

## 5 — Deliver the Legal Hold Report

```
## Arctera Legal Hold Report — [Matter Name]

**Matter:** [name]
**Custodians in scope:** [list]
**Keywords in scope:** [list]
**Date range:** [range]
**Archives searched:** Email, Files, Collaboration

**Items already on hold:** [N]
**New hold candidates — High priority:** [N]
**New hold candidates — Medium priority:** [N]
**Items out of scope:** [N]

---

### Items Already on Legal Hold

| # | Archive | Date | Custodian | Subject / Filename / Chat Room | Hold Tag |
|---|---------|------|-----------|-------------------------------|----------|

---

### New Hold Candidates — High Priority

| # | Archive | Date | Custodian | Subject / Filename / Chat Room | Reason for Hold | Key Excerpt |
|---|---------|------|-----------|-------------------------------|-----------------|-------------|

---

### New Hold Candidates — Medium Priority

| # | Archive | Date | Custodian | Subject / Filename / Chat Room | Reason | Excerpt |
|---|---------|------|-----------|-------------------------------|--------|---------|

---

### Hold Coverage by Custodian

| Custodian | Already Held | New Candidates | Gap? |
|-----------|-------------|----------------|------|

---

### Recommended Actions

- [ ] Place item #N (mailID: ...) on legal hold — High priority — contains [reason]
- [ ] Verify hold coverage for custodian [name] — [N] items not yet held
- [ ] Review medium-priority candidates with legal counsel before placing on hold
- [ ] Re-run `/arctera-legal-hold` after hold placement to confirm coverage
- [ ] Run `/arctera-custodian <name>` for a complete communication pull on each custodian
```
