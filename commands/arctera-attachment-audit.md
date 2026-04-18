You are running the **Arctera Attachment Audit** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **File type(s)** — attachment extension(s) to audit (e.g. "xlsx", "pdf", "zip", "docx")
- **Sender** — optional: specific sender to focus on (e.g. "andrew.fastow@enron.com")
- **Recipient** — optional: specific recipient or domain
- **Date range** — optional scope
- **Content filter** — optional: specific filename keyword or subject

If no file type is specified, ask:
> *"What attachment type(s) would you like to audit? (e.g. pdf, xlsx, zip, docx, ppt)"*

---

## 2 — Build attachment queries

**By file extension:**
```
ATTACHMENTS.EXTENSION:<ext> [AND SENDER:"sender@domain.com"] [AND FROMORTO:"recipient@domain.com"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

For multiple extensions:
```
(ATTACHMENTS.EXTENSION:pdf OR ATTACHMENTS.EXTENSION:xlsx OR ATTACHMENTS.EXTENSION:zip) [AND SENDER:...] [AND MAILDATE:...]
```

**By filename keyword (if provided):**
```
ATTACHMENTS.FILENAME:"<keyword>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Combined with content filter (if provided):**
```
ATTACHMENTS.EXTENSION:<ext> AND ENTIREMESSAGE:"<content keyword>" [AND MAILDATE:...]
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search email archive

Attachment metadata is primarily in the **email** archive. Use `search_email`:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0`

Paginate while `has_more` is true, up to 100 results.

Also search `search_file` for any standalone files matching the content filter.

---

## 4 — Triage and classify

For each result, note:
- **Sender** and **recipient(s)**
- **Attachment name** (from snippet or metadata)
- **External recipients**: flag any non-corporate domain recipients
- **Sensitive content markers**: "confidential", "restricted", "financial", "PII"
- **Volume**: sender with unusually high attachment volume

For the top 10 most significant items, call `get_email(mailID)` to retrieve full content.
Only use IDs from this session.

---

## 5 — Deliver the Attachment Audit Report

```
## Arctera Attachment Audit Report

**File type(s) audited:** [list]
**Sender filter:** [email or "all senders"]
**Recipient filter:** [email/domain or "all recipients"]
**Date range:** [range or "all dates"]
**Total emails with matching attachments:** [N]

---

### Attachment Inventory

| # | Date | Sender | Recipient(s) | Attachment Name | File Type | External Recipient? | Sensitive Content? |
|---|------|--------|--------------|-----------------|-----------|---------------------|--------------------|

---

### External Transmissions (attachments sent outside corporate boundary)

| # | Date | Sender | External Recipient | Attachment Name | Risk Level |
|---|------|--------|-------------------|-----------------|------------|

---

### Top Senders by Attachment Volume

| Sender | Total Attachments | External Sends | Most Common Type |
|--------|------------------|----------------|-----------------|

---

### Recommended Next Steps

- [ ] Investigate item #N — [file type] sent to external domain on [date]
- [ ] Run `/arctera-custodian <sender>` for a full communication pull on high-volume senders
- [ ] Run `/arctera-data-exposure` if external attachment transmissions suggest data leakage
- [ ] Run `/arctera-audit-pii` if attachments may contain PII before any export
```
