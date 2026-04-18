You are running the **Arctera Point-to-Point Communication Audit** workflow.

User request: $ARGUMENTS

---

## 1 — Identify both parties from $ARGUMENTS

Extract:
- **Party A** — first person's email address or name (e.g. "ken.lay@enron.com")
- **Party B** — second person's email address or name (e.g. "jeff.skilling@enron.com")
- **Date range** — optional scope
- **Topic filter** — optional keyword to narrow the search
- **Archive scope** — default: all three archives

If both parties cannot be identified, stop and ask:
> *"Please provide the email addresses or names of both parties whose communications you want to review."*

---

## 2 — Build bidirectional queries

For **email**, capture both directions of the conversation:
```
(SENDER:"partyA@domain.com" AND FROMORTO:"partyB@domain.com") OR (SENDER:"partyB@domain.com" AND FROMORTO:"partyA@domain.com") [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]] [AND ENTIREMESSAGE:"keyword"]
```

For **files**, search by both names:
```
ENTIREMESSAGE:"Party A Name" AND ENTIREMESSAGE:"Party B Name" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

For **collaboration**, search by sender in both directions:
```
(FROMORTO:"partyA@domain.com" OR FROMORTO:"partyB@domain.com") AND ENTIREMESSAGE:"Party B Name" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all applicable archives in parallel

Call `search_email`, `search_file`, and `search_message` simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — to reconstruct the relationship timeline)
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 results per archive.

---

## 4 — Triage and preview

From combined results, identify:
1. Communication frequency and date range (first to last contact)
2. Topics discussed across their communications
3. Third parties included in their communications (potential undisclosed relationships)
4. Any items on legal hold
5. Tone and pattern — escalation, urgency, unusual communication patterns

Preview the top 10 most significant items. Only use IDs from this session.

---

## 5 — Deliver the Point-to-Point Report

```
## Arctera Communication Audit — [Party A] ↔ [Party B]

**Party A:** [name and email]
**Party B:** [name and email]
**Date range:** [range or "all available dates"]
**Topic filter:** [keyword or "none — full pull"]
**Archives searched:** [Email / Files / Collab / All]

**Total communications:** [N] ([n] emails, [n] files, [n] collab)
**Date range of contact:** [first date] to [last date]
**Items on legal hold:** [N]

---

### Communication Timeline

| # | Date | Archive | Initiated by | Subject / Filename / Room | Key Excerpt | Flag |
|---|------|---------|--------------|--------------------------|-------------|------|

---

### Relationship Summary

| Metric | Value |
|---|---|
| Total communications | N |
| Emails from Party A to Party B | N |
| Emails from Party B to Party A | N |
| First contact | date |
| Last contact | date |
| Third parties included | [names] |
| Topics discussed | [key themes] |

---

### Notable Items

*(Top 3–5 most significant communications with full excerpt)*

---

### Recommended Next Steps

- [ ] Full review of item #N — [reason it is notable]
- [ ] Run `/arctera-timeline` to map this relationship against corporate events
- [ ] Run `/arctera-matter` if this relationship is relevant to a named legal matter
- [ ] Flag third party [name] included in [N] communications for further investigation
```
