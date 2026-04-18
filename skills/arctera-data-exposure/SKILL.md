---
name: arctera-data-exposure
description: Detects sensitive data sent to external or unexpected recipients. Scans outbound email and file archives for confidential content, sensitive attachments, and communications to personal email domains — identifying potential data exfiltration, unauthorised disclosure, or accidental leakage.
type: skill
---

# Skill: Arctera Sensitive Data External Exposure Detection

## Purpose
Identify sensitive organisational data that was sent outside the corporate boundary — to personal email accounts, unknown external domains, competitors, or unauthorised third parties. Covers confidential documents, PII-containing communications, restricted attachments, and any outbound communication bearing sensitive content markers. Produces a risk-tiered exposure report with recommended investigation actions.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails (primary for outbound) | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |

*Note: `search_message` (collaboration) is not typically used for outbound exposure detection unless the user specifies external chat participants.*

---

## Step 1 — Parse Exposure Parameters

Extract from the user's input:

| Parameter | How to identify | Required? |
|---|---|---|
| Internal domain | Corporate domain(s) to define "internal" | Required |
| Content filter | Type of data to focus on | Optional — default: all sensitive markers |
| Date range | "from … to", year references | Optional |
| Specific sender | Custodian to focus on | Optional |

If no internal domain can be identified, stop and ask:
> *"What is the corporate email domain to use as the internal boundary (e.g. enron.com)?"*

---

## Step 2 — Build Exposure Detection Queries

Run all four queries against email; run Query 2 against files as well.

**Query 1 — Confidential content markers:**
```
(ENTIREMESSAGE:confidential OR ENTIREMESSAGE:"do not share" OR ENTIREMESSAGE:proprietary OR ENTIREMESSAGE:"internal only" OR ENTIREMESSAGE:restricted) AND SENDER:<internaldomain>* [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 2 — Legal / privileged markers:**
```
(ENTIREMESSAGE:"attorney-client" OR ENTIREMESSAGE:"privileged" OR ENTIREMESSAGE:"work product" OR ENTIREMESSAGE:"under seal") AND SENDER:<internaldomain>* [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 3 — Sensitive attachments sent outbound:**
```
(ATTACHMENTS.EXTENSION:pdf OR ATTACHMENTS.EXTENSION:xlsx OR ATTACHMENTS.EXTENSION:docx OR ATTACHMENTS.EXTENSION:zip) AND SENDER:<internaldomain>* [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 4 — Personal email domains as recipients (exfiltration signal):**
```
(FROMORTO:gmail.com OR FROMORTO:yahoo.com OR FROMORTO:hotmail.com OR FROMORTO:icloud.com OR FROMORTO:outlook.com) AND SENDER:<internaldomain>* [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call `search_email` with all four queries and `search_file` with Query 2, all **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file`

Paginate while `has_more` is true, up to 100 results per query.

---

## Step 4 — Classify Each Result

**Recipient Type:**
| Type | Criteria |
|---|---|
| Personal email | gmail, yahoo, hotmail, icloud, outlook.com personal |
| Unknown external | Non-corporate domain not in an approved list |
| External counsel | Recognised law firm domain |
| Regulator | SEC, FINRA, FCA, DOJ, CFTC, etc. |
| Known business partner | Approved counterparty domain |

**Exposure Risk:**
| Risk Level | Criteria |
|---|---|
| **High** | Confidential/privileged content OR attachment sent to personal email or unknown external |
| **Medium** | Sensitive content sent to known external (counsel, partner) — may be authorised but needs confirmation |
| **Low** | Standard business communication with sensitive marker to known counterparty |

For **High-risk** items, call the preview tool to confirm the exposure — using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`

Do not preview more than 10 items without checking with the user.

---

## Step 5 — Produce the Data Exposure Report

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

### High-Risk Exposures — Immediate Investigation Required

| # | Archive | Date | Sender | External Recipient / Domain | Content Signal | Attachment? | Key Excerpt |
|---|---------|------|--------|----------------------------|----------------|-------------|-------------|

---

### Medium-Risk Exposures — Review Recommended

| # | Archive | Date | Sender | External Recipient | Content Signal | Excerpt |
|---|---------|------|--------|-------------------|----------------|---------|

---

### Exposure by Recipient Type

| Recipient Type | Total | High Risk | Medium Risk | Low Risk |
|---|---|---|---|---|
| Personal email | N | N | N | N |
| Unknown external | N | N | N | N |
| External counsel | N | N | N | N |
| Regulator | N | N | N | N |
| Known business partner | N | N | N | N |

---

### Top Senders (by exposure volume)

| Sender | High-Risk Items | Medium-Risk Items | Top External Recipients |
|--------|----------------|-------------------|------------------------|

---

### Recommended Actions

- [ ] Investigate item #N — [content type] sent to [personal/unknown domain] on [date]
- [ ] Review custodian [name] — [N] high-risk outbound items — run `/arctera-custodian`
- [ ] Confirm item #N was an authorised disclosure to external counsel
- [ ] Escalate high-risk exposures to Information Security and Legal immediately
- [ ] Run `/arctera-audit-pii` to check if any exposed content contains PII
- [ ] Consider access control review for the top sending custodians
```

---

## Behaviour Rules

- Run all four exposure queries **in parallel** across email; run Query 2 against files simultaneously.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- Do not reproduce raw PII found in previewed content — describe the data type.
- If a recipient domain is ambiguous, classify as "Unknown external" and flag for human review.
- Do not call more than 10 get tools without checking with the user.
