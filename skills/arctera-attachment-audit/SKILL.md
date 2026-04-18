---
name: arctera-attachment-audit
description: Audits attachments in the email archive by file type, sender, recipient, or filename keyword. Produces an attachment inventory identifying what was shared, with whom, and whether any attachments were sent to external parties — useful for data exfiltration review, IP protection, and document inventory.
type: skill
---

# Skill: Arctera Attachment Audit

## Purpose
Produce a comprehensive inventory of email attachments matching specified criteria — file type, sender, recipient, date range, or filename keyword. Identifies what documents were shared, who shared them, whether they went to external recipients, and whether any sensitive content markers are present. Used in data exfiltration investigations, intellectual property reviews, pre-litigation document identification, and compliance audits.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails (primary — attachment metadata) | `get_email` | `mailID` |
| `search_file` | Files / Documents (supplementary) | `get_file` | `fileId`, `batchId`, `messageId` |

*`search_message` is not used for attachment audits unless the user specifies file sharing in chat.*

---

## Step 1 — Parse Attachment Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| File type(s) | Extension: pdf, xlsx, docx, zip, ppt, csv, eml | Required — ask if not provided |
| Sender | Email address or domain | Optional |
| Recipient | Email address or domain | Optional |
| Date range | Year or date range | Optional |
| Filename keyword | Specific filename terms | Optional |
| Content filter | Topic keyword in email body | Optional |

If no file type is specified, stop and ask:
> *"What attachment type(s) would you like to audit? (e.g. pdf, xlsx, zip, docx, ppt)"*

---

## Step 2 — Build Attachment Queries

**By file extension:**
```
ATTACHMENTS.EXTENSION:<ext> [AND SENDER:"sender@domain.com"] [AND FROMORTO:"recipient@domain.com"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

For multiple extensions:
```
(ATTACHMENTS.EXTENSION:pdf OR ATTACHMENTS.EXTENSION:xlsx OR ATTACHMENTS.EXTENSION:zip) [AND SENDER:...] [AND MAILDATE:...]
```

**By filename keyword:**
```
ATTACHMENTS.FILENAME:"<keyword>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Combined with content filter:**
```
ATTACHMENTS.EXTENSION:<ext> AND ENTIREMESSAGE:"<content keyword>" [AND SENDER:...] [AND MAILDATE:...]
```

**Personal email domains (exfiltration signal):**
```
ATTACHMENTS.EXTENSION:<ext> AND (FROMORTO:gmail.com OR FROMORTO:yahoo.com OR FROMORTO:hotmail.com) [AND SENDER:<internaldomain>*] [AND MAILDATE:...]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches

Call `search_email` with all applicable queries:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0`

Also call `search_file` with the content filter query if provided:
- `contentDetails: 1`
- `pageSize: 20`
- `pageNum: 1`

Run email and file searches **in parallel**.
Paginate while `has_more` is true, up to 100 results per query.

---

## Step 4 — Classify Each Result

For each result, note:

**Recipient Type:**
| Type | Criteria |
|---|---|
| Internal only | All recipients on corporate domain |
| External — known | Law firm, regulator, approved counterparty |
| External — personal | gmail, yahoo, hotmail, icloud |
| External — unknown | Any non-corporate domain not in approved list |

**Risk Level:**
| Level | Criteria |
|---|---|
| High | Attachment sent to personal email or unknown external domain |
| Medium | Attachment with sensitive markers sent to known external |
| Low | Internal send or known external with no risk signals |

For **High-risk** items, call `get_email(mailID)` to retrieve full content and confirm the attachment details — using **IDs from this session only**.

---

## Step 5 — Produce the Attachment Audit Report

```
## Arctera Attachment Audit Report

**File type(s) audited:** [list]
**Sender filter:** [email or "all senders"]
**Recipient filter:** [email/domain or "all recipients"]
**Date range:** [range or "all dates"]
**Content filter:** [keyword or "none"]

**Total emails with matching attachments:** [N]
**High-risk transmissions:** [N]
**Medium-risk transmissions:** [N]
**Internal only:** [N]

---

### Attachment Inventory

| # | Date | Sender | Recipient(s) | Attachment Name | File Type | External? | Risk |
|---|------|--------|--------------|-----------------|-----------|-----------|------|

---

### High-Risk — Attachments Sent to Personal or Unknown External

| # | Date | Sender | External Recipient | Attachment Name | Content Signal | Action |
|---|------|--------|--------------------|-----------------|----------------|--------|

---

### External Transmissions Summary

| Recipient Type | Count | Senders | Most Recent |
|---|---|---|---|
| Personal email | N | [names] | date |
| Unknown external | N | [names] | date |
| External counsel | N | [names] | date |
| Regulator | N | [names] | date |

---

### Top Senders by Attachment Volume

| Sender | Total Attachments | External Sends | Personal Email Sends | Most Common Type |
|--------|------------------|----------------|----------------------|-----------------|

---

### Recommended Next Steps

- [ ] Investigate item #N — [type] sent to [personal/unknown domain] on [date]
- [ ] Run `/arctera-custodian <sender>` for top external senders
- [ ] Run `/arctera-data-exposure` for a broader sensitive-content exposure scan
- [ ] Run `/arctera-audit-pii` if any attachments may contain PII before export
```

---

## Behaviour Rules

- Email attachment metadata is in `search_email` — always use this as the primary tool.
- Run all queries **in parallel** where multiple queries apply.
- Only use get tools with IDs from the **current session**.
- For `get_email` results, report attachment names and content type — do not display raw HTML.
- Flag every transmission to a personal email domain (gmail, yahoo, hotmail, icloud, outlook.com personal) as High-risk.
- Do not call more than 10 get tools without checking with the user.
