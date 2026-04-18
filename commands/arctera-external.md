You are running the **Arctera External Communication Audit** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Internal domain(s)** — the corporate domain(s) to define "internal" (e.g. "enron.com")
- **Date range** — optional scope
- **Custodian** — optional specific sender to focus on
- **Content filter** — optional: keyword to focus on (e.g. "confidential", "financial")
- **Domain allowlist** — optional: known-approved external domains to exclude from flagging

If no internal domain can be identified, ask:
> *"What is the corporate email domain? (e.g. enron.com) I need this to identify what counts as 'external'."*

---

## 2 — Build outbound queries

**Query 1 — All outbound from internal domain:**
```
SENDER:<internaldomain>* AND NOT FROMORTO:<internaldomain>* [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 2 — Outbound with sensitive content markers:**
```
SENDER:<internaldomain>* AND (ENTIREMESSAGE:confidential OR ENTIREMESSAGE:proprietary OR ENTIREMESSAGE:restricted) [AND MAILDATE:...]
```

**Query 3 — Outbound with attachments:**
```
SENDER:<internaldomain>* AND ATTACHMENTS.EXTENSION:pdf [AND MAILDATE:...]
```

**Query 4 — Outbound to personal email domains:**
```
SENDER:<internaldomain>* AND (FROMORTO:gmail.com OR FROMORTO:yahoo.com OR FROMORTO:hotmail.com OR FROMORTO:icloud.com) [AND MAILDATE:...]
```

Append custodian filter as `SENDER:"custodian@corpdomain.com"` where provided.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search email archive

Use `search_email` for all four queries in parallel:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0`

Paginate while `has_more` is true, up to 100 results per query.

---

## 4 — Classify each result

**Recipient Category:**
- Personal email: gmail, yahoo, hotmail, icloud, outlook.com (personal)
- External counsel: recognised law firm domain
- Regulator: SEC, FINRA, FCA, DOJ, etc.
- Approved counterparty: known business partner (if allowlist provided)
- Unknown external: any other non-corporate domain

**Risk Level:**
- High: sensitive content or attachments to personal email or unknown external
- Medium: any outbound with attachments to known external
- Low: standard business communication to known counterparty

For High-risk items, call `get_email(mailID)`. Only use IDs from this session.

---

## 5 — Deliver the External Communication Audit Report

```
## Arctera External Communication Audit

**Internal domain:** [domain]
**Custodian filter:** [email or "all internal senders"]
**Date range:** [range or "all dates"]
**Content filter:** [keyword or "all outbound"]

**Total outbound communications found:** [N]
**High-risk:** [N] | **Medium-risk:** [N] | **Low-risk:** [N]

---

### High-Risk External Communications

| # | Date | Sender | External Recipient | Recipient Type | Content | Attachment? | Excerpt |
|---|------|--------|--------------------|----------------|---------|-------------|---------|

---

### External Communication by Recipient Type

| Recipient Type | Count | Senders | Date Range |
|---|---|---|---|
| Personal email | N | [names] | [range] |
| Unknown external | N | [names] | [range] |
| External counsel | N | [names] | [range] |
| Regulator | N | [names] | [range] |
| Approved counterparty | N | [names] | [range] |

---

### Top External Senders

| Sender | Total External | High-Risk | Top External Domain |
|--------|---------------|-----------|---------------------|

---

### Recommended Next Steps

- [ ] Investigate item #N — [content] sent to [personal/unknown domain] on [date]
- [ ] Run `/arctera-custodian <sender>` for top external senders
- [ ] Run `/arctera-data-exposure` for a focused sensitive-content scan
- [ ] Confirm item #N was authorised disclosure to external counsel or regulator
```
