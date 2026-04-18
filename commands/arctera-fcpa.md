You are running the **Arctera FCPA & Anti-Bribery Compliance Scan** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Archive scope** — "emails", "files", "chat", or "all" (default: all)
- **Date range** — optional scope
- **Custodians or third parties** — optional parties to focus on (agents, consultants, government-linked contacts)
- **Geography** — optional country or region focus (e.g. "Nigeria", "India") for higher-risk jurisdictions

If no scope keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for FCPA and anti-bribery signals?"*

---

## 2 — Build FCPA signal queries

```
ENTIREMESSAGE:"facilitation payment" OR ENTIREMESSAGE:kickback OR ENTIREMESSAGE:bribery OR ENTIREMESSAGE:bribe
ENTIREMESSAGE:"government official" OR ENTIREMESSAGE:"public official" OR ENTIREMESSAGE:"foreign official"
ENTIREMESSAGE:"gifts and entertainment" OR ENTIREMESSAGE:"hospitality" OR ENTIREMESSAGE:"gift policy"
ENTIREMESSAGE:"agent" OR ENTIREMESSAGE:"intermediary" OR ENTIREMESSAGE:"third-party representative"
ENTIREMESSAGE:"commission" OR ENTIREMESSAGE:"success fee" OR ENTIREMESSAGE:"finder's fee"
ENTIREMESSAGE:"cash payment" OR ENTIREMESSAGE:"off the books" OR ENTIREMESSAGE:"undisclosed payment"
ENTIREMESSAGE:"FCPA" OR ENTIREMESSAGE:"UK Bribery Act" OR ENTIREMESSAGE:"anti-corruption"
```

If a geography is specified:
```
ENTIREMESSAGE:"<country/region>" AND (ENTIREMESSAGE:official OR ENTIREMESSAGE:payment OR ENTIREMESSAGE:gift)
```

Append custodian and date filters with AND where provided.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search applicable archives in parallel

Call search tools simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 results per query.

---

## 4 — Classify each result

**FCPA Signal Type:**
- Direct bribery: explicit payment to government official
- Facilitation payment: payments to expedite routine government services
- Gifts & entertainment: potentially excessive hospitality to officials
- Third-party agent risk: high-commission agent in high-risk jurisdiction
- Anti-corruption policy: internal policy discussion or concern raised
- Self-disclosure: internal discussion of potential FCPA violation

**Risk Level:**
- Critical: explicit improper payment reference or self-disclosure
- High: agent/intermediary in high-risk jurisdiction with unusual compensation
- Medium: gifts/entertainment reference near government official interaction
- Low: general FCPA or anti-corruption policy reference

For Critical and High items, call the preview tool. Only use IDs from this session.

---

## 5 — Deliver the FCPA Risk Report

```
## Arctera FCPA & Anti-Bribery Risk Report

**Geography focus:** [country/region or "all jurisdictions"]
**Date range:** [range or "all dates"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Critical items:** [N] | **High:** [N] | **Medium:** [N]

---

### Critical Items — Explicit Improper Payment or Self-Disclosure

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-------------|

---

### High-Risk Items — Agent/Intermediary or Hospitality Concerns

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|---------|

---

### Risk Narrative

[2–3 paragraphs: nature of FCPA exposure, key parties, jurisdictions involved, recommended actions]

---

### Recommended Next Steps

- [ ] Engage outside counsel with FCPA expertise immediately for Critical items
- [ ] Preserve all flagged items — run `/arctera-legal-hold`
- [ ] Run `/arctera-custodian` on third-party agent parties identified
- [ ] Run `/arctera-between` for communications between [custodian] and [government contact]
- [ ] Consider voluntary disclosure assessment with outside counsel
```
