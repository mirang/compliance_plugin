You are running the **Arctera GDPR Erasure Pre-Deletion Audit** workflow.

User request: $ARGUMENTS

---

## 1 — Identify the individual from $ARGUMENTS

Extract:
- **Full name** — (e.g. "John Doe")
- **Email address** — corporate or personal (e.g. `john.doe@enron.com`)
- **Additional identifiers** — phone, employee ID, any other identifiers
- **Erasure basis** — optional context (e.g. "GDPR Article 17", "CCPA deletion", "no longer employed")

If neither a name nor an email can be identified, stop and ask:
> *"Please provide the individual's full name and/or email address to scope the erasure audit."*

---

## 2 — Build identification queries

Run the same multi-signal queries as a DSAR across all three archives:

```
FROMORTO:"individual@domain.com"
ENTIREMESSAGE:"First Last"
ENTIREMESSAGE:"Last, First"
ENTIREMESSAGE:"individual@domain.com"
```

Also include personal email and phone if known.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all three archives in parallel

Call `search_email`, `search_file`, and `search_message` simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 total results per archive.

---

## 4 — Classify each record by erasure eligibility

For every result, assign an erasure eligibility status:

| Status | Criteria |
|---|---|
| **Eligible for erasure** | Contains only personal data of the subject, no legal basis to retain |
| **Exempt — Legal hold** | `isLegalHold: true` — cannot be erased while hold is active |
| **Exempt — Regulatory** | Required by regulatory retention policy (e.g. SEC 17a-4, FINRA) |
| **Exempt — Legitimate interest** | Part of a business record with a legitimate retention basis |
| **Third-party data** | Contains data about other individuals — partial redaction may apply |
| **Needs review** | Cannot be automatically classified — requires human assessment |

---

## 5 — Preview exemption-flagged items

For items flagged as Legal hold or Regulatory exempt, call the preview tool to confirm:
- Email → `get_email(mailID)`
- File → `get_file(fileId, batchId, messageId)`
- Collab → `get_message(chatRoomId, collabMessageId, chatRoomName, matterId, collabDate)`

Only use IDs from this session.

---

## 6 — Deliver the Erasure Scope Report

```
## Arctera GDPR Erasure Audit — [Individual Name]

**Individual:** [Full name and email]
**Erasure basis:** [GDPR Article 17 / CCPA / other or "not specified"]
**Archives searched:** Email, Files, Collaboration
**Audit date:** [today's date]

**Total records identified:** [N] ([n] emails, [n] files, [n] collab)
**Eligible for erasure:** [N]
**Exempt — Legal hold:** [N]
**Exempt — Regulatory retention:** [N]
**Exempt — Legitimate interest:** [N]
**Needs human review:** [N]

---

### Erasure Eligibility Summary

| Archive | Total | Eligible | Legal Hold | Regulatory | Needs Review |
|---------|-------|----------|------------|------------|--------------|
| Email | N | N | N | N | N |
| Files | N | N | N | N | N |
| Collab | N | N | N | N | N |

---

### Records Eligible for Erasure

| # | Archive | Date | Subject / Filename / Room | Data Type | Reason Eligible |
|---|---------|------|--------------------------|-----------|-----------------|

---

### Records Exempt from Erasure

| # | Archive | Date | Subject / Filename / Room | Exemption Basis | Retention Period |
|---|---------|------|--------------------------|-----------------|-----------------|

---

### Records Requiring Human Review

| # | Archive | Date | Subject / Filename / Room | Reason for Review |
|---|---------|------|--------------------------|-------------------|

---

### Recommended Actions

- [ ] Process erasure for [N] eligible records — no exemption applies
- [ ] Do NOT erase [N] records on legal hold until hold is released
- [ ] Confirm regulatory retention period for [N] exempt records with compliance team
- [ ] Submit [N] records needing review to DPO / legal counsel for classification
- [ ] Log erasure action, date, and basis in the data subject request register
- [ ] Re-run audit after erasure to confirm completion
```

**Masking rule:** Never reproduce raw personal data values. Describe the data type found, not the value.
