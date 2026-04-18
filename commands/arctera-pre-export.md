You are running the **Arctera Pre-Export Compliance Checklist** workflow.

User request: $ARGUMENTS

---

## 1 — Define the export set from $ARGUMENTS

Extract the search criteria that define the production set to be exported:
- **Topic / matter name** — what the export covers
- **Custodians** — whose records are being exported
- **Date range** — the date scope of the export
- **Archive(s)** — which archives are included in the export

If the export set cannot be defined, ask:
> *"Please describe the export set — what topic, custodians, date range, and archives are included in this production?"*

---

## 2 — Run three parallel compliance checks

Execute all three checks **simultaneously** against the export set:

### Check 1 — PII Scan

Run the full PII query set against the export scope:
```
ENTIREMESSAGE:"social security" OR ENTIREMESSAGE:SSN [AND export filters]
ENTIREMESSAGE:"credit card" OR ENTIREMESSAGE:"card number" [AND export filters]
ENTIREMESSAGE:"date of birth" OR ENTIREMESSAGE:DOB [AND export filters]
ENTIREMESSAGE:"home address" OR ENTIREMESSAGE:"personal address" [AND export filters]
ENTIREMESSAGE:"account number" OR ENTIREMESSAGE:"routing number" [AND export filters]
ENTIREMESSAGE:"medical record" OR ENTIREMESSAGE:diagnosis [AND export filters]
```

### Check 2 — Legal Hold Status

```
CLASSIFICATION.TAGS:hold [AND export filters]
```

Check `isLegalHold` field in all results.

### Check 3 — Privilege Flag

```
CLASSIFICATION.TAGS:privilege [AND export filters]
ENTIREMESSAGE:"attorney-client" OR ENTIREMESSAGE:"work product" OR ENTIREMESSAGE:"privileged" [AND export filters]
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all applicable archives

For each check, call applicable search tools:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

---

## 4 — Classify results

For each result across all three checks, assign:
- **Check**: PII / Legal Hold / Privilege
- **Risk**: Critical (must resolve before export) / Review (human confirmation needed)
- **Action required**: Redact / Hold / Withhold / Review

For Critical items, call the preview tool to confirm. Only use IDs from this session.

---

## 5 — Deliver the Pre-Export Compliance Report

```
## Arctera Pre-Export Compliance Report

**Export set:** [topic / matter]
**Custodians:** [list]
**Date range:** [range]
**Archives in scope:** [list]
**Export check date:** [today's date]

---

### Compliance Check Summary

| Check | Status | Critical Items | Review Items |
|---|---|---|---|
| PII Scan | ⚠ FAIL / ✓ PASS | N | N |
| Legal Hold | ⚠ FAIL / ✓ PASS | N | N |
| Privilege | ⚠ FAIL / ✓ PASS | N | N |

**Overall export status:** ⚠ NOT CLEARED — [N] items require resolution before export

---

### Critical Items — Must Resolve Before Export

| # | Check | Archive | Date | Sender | Subject / Filename | Issue | Action |
|---|-------|---------|------|--------|--------------------|-------|--------|
| 1 | PII | Email | ... | ... | ... | SSN detected | Redact |
| 2 | Hold | File | ... | ... | ... | Legal hold active | Withhold |
| 3 | Privilege | Email | ... | ... | ... | Attorney-client | Withhold |

---

### Review Items — Human Confirmation Required

| # | Check | Archive | Date | Subject / Filename | Issue | Recommended Action |
|---|-------|---------|------|--------------------|-------|-------------------|

---

### Export Clearance Checklist

- [ ] Redact PII in [N] Critical items before including in production set
- [ ] Withhold [N] items on legal hold from this production — log as withheld
- [ ] Withhold [N] privilege-flagged items — generate privilege log entry for each
- [ ] Human review of [N] review items before final export decision
- [ ] Re-run `/arctera-pre-export` after redactions to confirm clearance
- [ ] Document export date, custodians, and withheld items in the production log
```

**PII masking rule:** Never reproduce raw PII values. Use masked format (SSN: `XXX-XX-####`).
