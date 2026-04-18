You are running the **Arctera Sensitive Data External Exposure** workflow.

User request: $ARGUMENTS

---

## 1 — Parse exposure parameters from $ARGUMENTS

Extract:
- **Internal domain(s)** — the corporate domain(s) (e.g. "enron.com") to distinguish internal from external
- **Content filter** — optional: type of sensitive data to look for (e.g. "confidential documents", "financial data", "PII")
- **Date range** — optional scope (e.g. "2001")
- **Specific sender** — optional custodian to focus on (e.g. "andrew.fastow@enron.com")

If no internal domain can be identified, ask:
> *"What is the corporate email domain to use as the internal boundary? (e.g. enron.com)"*

---

## 2 — Build exposure detection queries

**Query 1 — Confidential content sent externally (email):**
```
(ENTIREMESSAGE:confidential OR ENTIREMESSAGE:"do not share" OR ENTIREMESSAGE:proprietary OR ENTIREMESSAGE:"internal only") AND SENDER:<internal-domain>* [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 2 — Documents with sensitive labels:**
```
(ENTIREMESSAGE:"restricted" OR ENTIREMESSAGE:"for internal use" OR ENTIREMESSAGE:"attorney-client") AND SENDER:<internal-domain>* [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 3 — Attachments sent outbound:**
```
ATTACHMENTS.EXTENSION:pdf OR ATTACHMENTS.EXTENSION:xlsx OR ATTACHMENTS.EXTENSION:docx [AND SENDER:<internal-domain>*] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 4 — Personal email domains as recipients (high-risk exfiltration indicator):**
```
FROMORTO:gmail.com OR FROMORTO:yahoo.com OR FROMORTO:hotmail.com OR FROMORTO:icloud.com [AND SENDER:<internal-domain>*] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search email and files in parallel

Email is the primary archive for outbound communication. Also search files for externally shared documents.
Use `search_email` and `search_file` in parallel:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files

---

## 4 — Classify each result

For every result, assign:

**Recipient Type:**
| Type | Criteria |
|---|---|
| Personal email | gmail, yahoo, hotmail, icloud, outlook.com personal |
| Competitor domain | Known competitor domains |
| Law firm / external counsel | Recognised legal domain |
| Regulator | SEC, FINRA, FCA, DOJ, etc. |
| Unknown external | Any non-corporate domain not matching above |

**Exposure Risk:**
| Risk | Criteria |
|---|---|
| High | Confidential content sent to personal email or unknown external |
| Medium | Sensitive attachment sent to known external party without clear authorisation |
| Low | Standard business communication to external party, no sensitive markers |

For **High-risk** items, call the preview tool to confirm:
- Email → `get_email(mailID)`
- File → `get_file(fileId, batchId, messageId)`

Only use IDs from this session.

---

## 5 — Deliver the Data Exposure Report

```
## Arctera Data Exposure Report

**Internal domain(s):** [domain]
**Content filter:** [type or "all sensitive markers"]
**Date range:** [range or "all dates"]
**Archives searched:** Email, Files
**Queries run:** [N]

**Total records reviewed:** [N] ([n] emails, [n] files)
**High-risk exposures:** [N]
**Medium-risk exposures:** [N]
**Low-risk / baseline:** [N]

---

### High-Risk Exposures — Immediate Review Required

| # | Archive | Date | Sender | Recipient / Domain | Attachment | Content Signal | Key Excerpt |
|---|---------|------|--------|-------------------|------------|----------------|-------------|

---

### Medium-Risk Exposures — Review Recommended

| # | Archive | Date | Sender | Recipient / Domain | Content Signal | Excerpt |
|---|---------|------|--------|-------------------|----------------|---------|

---

### Exposure by Recipient Type

| Recipient Type | Count | High Risk | Medium Risk |
|---|---|---|---|
| Personal email | N | N | N |
| Unknown external | N | N | N |
| External counsel | N | N | N |
| Regulator | N | N | N |

---

### Recommended Actions

- [ ] Investigate item #N — confidential content sent to personal email on [date]
- [ ] Review sender [name]'s outbound communications for pattern of exfiltration
- [ ] Confirm item #N was an authorised disclosure to external counsel
- [ ] Run `/arctera-custodian <sender>` for a full communication pull on high-risk senders
- [ ] Escalate High-risk items to Information Security and Legal immediately
```
