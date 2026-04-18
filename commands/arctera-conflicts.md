You are running the **Arctera Conflicts of Interest Detection** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Archive scope** — "emails", "files", "chat", or "all" (default: all)
- **Date range** — optional scope
- **Custodians** — optional specific parties to investigate
- **Matter or transaction** — optional specific deal or matter to scope the search

If no scope keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for conflicts of interest?"*

---

## 2 — Build conflicts signal queries

```
ENTIREMESSAGE:"conflict of interest" OR ENTIREMESSAGE:"conflicts of interest"
ENTIREMESSAGE:recuse OR ENTIREMESSAGE:recusal OR ENTIREMESSAGE:"step aside"
ENTIREMESSAGE:"undisclosed" OR ENTIREMESSAGE:"personal interest" OR ENTIREMESSAGE:"financial interest"
ENTIREMESSAGE:"related party" OR ENTIREMESSAGE:"arm's length" OR ENTIREMESSAGE:"arm-length"
ENTIREMESSAGE:"dual role" OR ENTIREMESSAGE:"wearing two hats" OR ENTIREMESSAGE:"self-dealing"
ENTIREMESSAGE:"outside business" OR ENTIREMESSAGE:"outside interest" OR ENTIREMESSAGE:"secondary employment"
ENTIREMESSAGE:"disclosure form" OR ENTIREMESSAGE:"COI form" OR ENTIREMESSAGE:"conflict disclosure"
```

Append custodian and date filters with AND where provided.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search applicable archives in parallel

Call search tools simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

---

## 4 — Classify each result

**Conflict Type:**
- Undisclosed financial interest: personal financial stake not disclosed
- Self-dealing: party on both sides of a transaction
- Recusal failure: evidence a party participated who should have recused
- Outside business activity: undisclosed external business interest
- Disclosure process: COI form, disclosure discussion (may be benign)

**Risk Level:**
- Critical: undisclosed conflict with material financial impact
- High: evidence of participation where recusal was required
- Medium: reference to conflict concern without clear resolution
- Low: general COI policy or disclosure process reference

For Critical and High items, call the preview tool. Only use IDs from this session.

---

## 5 — Deliver the Conflicts Report

```
## Arctera Conflicts of Interest Report

**Date range:** [range or "all dates"]
**Custodians in scope:** [list or "all"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Critical items:** [N] | **High:** [N] | **Medium:** [N]

---

### Critical Items — Undisclosed Conflict or Self-Dealing

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Conflict Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|---------------|-------------|

---

### High-Risk Items — Recusal or Participation Concerns

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Conflict Type | Excerpt |
|---|---------|------|-----------------|--------------------------|---------------|---------|

---

### Risk Narrative

[2–3 paragraphs: nature of conflicts identified, parties involved, transactions affected]

---

### Recommended Next Steps

- [ ] Escalate Critical items to General Counsel and Audit Committee
- [ ] Verify COI disclosure records for custodians named in Critical items
- [ ] Run `/arctera-between` for parties on both sides of flagged transactions
- [ ] Preserve flagged items — run `/arctera-legal-hold` if litigation risk is elevated
```
