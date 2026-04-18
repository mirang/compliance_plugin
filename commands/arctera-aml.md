You are running the **Arctera Anti-Money Laundering (AML) Signal Scan** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Archive scope** — "emails", "files", "chat", or "all" (default: all)
- **Date range** — optional scope
- **Custodians or counterparties** — optional parties to focus on

If no scope keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for AML signals?"*

---

## 2 — Build AML signal queries

```
ENTIREMESSAGE:"money laundering" OR ENTIREMESSAGE:structuring OR ENTIREMESSAGE:"smurfing"
ENTIREMESSAGE:"suspicious activity" OR ENTIREMESSAGE:SAR OR ENTIREMESSAGE:"suspicious transaction"
ENTIREMESSAGE:"wire transfer" OR ENTIREMESSAGE:"cash transaction" OR ENTIREMESSAGE:"cash deposit"
ENTIREMESSAGE:"beneficial owner" OR ENTIREMESSAGE:"ultimate beneficial owner" OR ENTIREMESSAGE:UBO
ENTIREMESSAGE:"shell company" OR ENTIREMESSAGE:"offshore account" OR ENTIREMESSAGE:"nominee"
ENTIREMESSAGE:"know your customer" OR ENTIREMESSAGE:KYC OR ENTIREMESSAGE:"customer due diligence" OR ENTIREMESSAGE:CDD
ENTIREMESSAGE:IBAN OR ENTIREMESSAGE:"routing number" OR ENTIREMESSAGE:"account number" OR ENTIREMESSAGE:SWIFT
ENTIREMESSAGE:"layering" OR ENTIREMESSAGE:"placement" OR ENTIREMESSAGE:"integration" [in financial context]
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

**AML Signal Type:**
- Structuring / smurfing: breaking up transactions to avoid reporting thresholds
- Suspicious activity: SAR reference, explicit suspicion flagged
- Beneficial ownership: UBO / shell company / nominee discussion
- KYC/CDD gap: due diligence concern or gap identified
- Unusual wire / cash: large or unusual wire transfers or cash transactions
- Layering: complex multi-entity transaction structure discussions

**Risk Level:**
- Critical: explicit SAR filing, structuring, or money laundering reference
- High: beneficial ownership opacity or unusual transaction structure concern
- Medium: KYC/CDD gap or due diligence concern
- Low: general AML compliance reference (policy, training)

For Critical and High items, call the preview tool. Only use IDs from this session.
Do not reproduce any raw account numbers or financial identifiers — describe the type only.

---

## 5 — Deliver the AML Risk Report

```
## Arctera AML Signal Report

**Date range:** [range or "all dates"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Critical items:** [N] | **High:** [N] | **Medium:** [N]

---

### Critical Items — SAR, Structuring, or Explicit AML Reference

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-------------|

---

### High-Risk Items — Beneficial Ownership or Transaction Structure Concerns

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|---------|

---

### Risk Narrative

[2–3 paragraphs: nature of AML risk, key parties, transaction patterns, recommended escalation]

---

### Recommended Next Steps

- [ ] Escalate Critical items to Chief Compliance Officer and AML team immediately
- [ ] File or review SAR obligations for Critical items with compliance team
- [ ] Preserve flagged items — run `/arctera-legal-hold`
- [ ] Run `/arctera-custodian` on parties involved in suspicious transactions
- [ ] Engage outside AML counsel for Critical items
```

**Masking rule:** Never reproduce raw account numbers, routing numbers, IBAN, or financial identifiers. Describe the type of financial identifier found.
