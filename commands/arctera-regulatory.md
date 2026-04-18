You are running the **Arctera Regulatory Red Flag Scan** workflow.

User request: $ARGUMENTS

---

## 1 — Parse regulatory parameters from $ARGUMENTS

Extract:
- **Regulator(s)** — optional specific regulator to focus on (e.g. "SEC", "FINRA", "FCA", "CFTC", "DOJ")
- **Archive scope** — "emails", "files", "chat", or "all" (default: all)
- **Date range** — optional (e.g. "2001")
- **Custodians** — optional specific parties to focus on

If no scope keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for regulatory red flags?"*

---

## 2 — Build regulatory signal queries

Run all applicable queries across the chosen archives:

```
ENTIREMESSAGE:SEC OR ENTIREMESSAGE:FINRA OR ENTIREMESSAGE:CFTC OR ENTIREMESSAGE:FCA OR ENTIREMESSAGE:DOJ OR ENTIREMESSAGE:FERC
ENTIREMESSAGE:subpoena OR ENTIREMESSAGE:"regulatory inquiry" OR ENTIREMESSAGE:"government investigation"
ENTIREMESSAGE:"cease and desist" OR ENTIREMESSAGE:"Wells notice" OR ENTIREMESSAGE:"enforcement action"
ENTIREMESSAGE:"examination" OR ENTIREMESSAGE:"regulatory review" OR ENTIREMESSAGE:"regulatory audit"
ENTIREMESSAGE:"consent order" OR ENTIREMESSAGE:"deferred prosecution" OR ENTIREMESSAGE:settlement
ENTIREMESSAGE:"whistleblower" OR ENTIREMESSAGE:"qui tam" OR ENTIREMESSAGE:"False Claims Act"
```

If a specific regulator is named in $ARGUMENTS, prioritise that regulator's query first.
Append custodian and date filters with AND where provided.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search applicable archives in parallel

Call applicable search tools simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 results per query.

---

## 4 — Classify each result

**Regulatory Signal Type:**
- Active inquiry: subpoena, investigation notice, examination letter, Wells notice
- Enforcement action: consent order, settlement, deferred prosecution
- Internal awareness: internal discussion of regulatory risk
- Whistleblower: any mention of whistleblower, qui tam, or protected disclosure
- General compliance: compliance review, audit, examination references

**Risk Level:**
- Critical: active regulator contact or enforcement reference
- High: internal discussion of regulatory investigation or anticipated inquiry
- Medium: general compliance/audit reference in relevant context
- Low: generic regulatory term with no surrounding risk context

For Critical and High items, call the preview tool (`get_email`, `get_file`, or `get_message`) to confirm.
Only use IDs from this session.

---

## 5 — Deliver the Regulatory Risk Report

```
## Arctera Regulatory Red Flag Report

**Regulator focus:** [specific regulator or "all regulators"]
**Date range:** [range or "all dates"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Total records found:** [N] ([n] emails, [n] files, [n] collab)
**Critical items:** [N] | **High:** [N] | **Medium:** [N] | **Low:** [N]

---

### Critical Items — Active Regulatory Contact or Enforcement

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-------------|

---

### High-Risk Items — Internal Regulatory Risk Discussion

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|---------|

---

### Risk Narrative

[2–3 paragraphs: nature of regulatory exposure, key parties involved, timeline, recommended escalation path]

---

### Recommended Next Steps

- [ ] Escalate item #N (active regulator contact) to General Counsel immediately
- [ ] Preserve all Critical and High-risk items — run `/arctera-legal-hold`
- [ ] Run `/arctera-custodian` on parties named in regulatory communications
- [ ] Run `/arctera-audit-pii` before any regulatory production
- [ ] Engage outside counsel for items referencing [regulator]
```
