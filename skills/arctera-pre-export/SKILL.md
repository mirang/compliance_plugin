---
name: arctera-pre-export
description: Pre-export compliance checklist that runs three parallel checks — PII detection, legal hold identification, and privilege flagging — against a defined production set before it is exported. Produces a PASS/FAIL clearance report with action items for each failed check.
type: skill
---

# Skill: Arctera Pre-Export Compliance Checklist

## Purpose
Before any production set is exported or produced — in litigation, regulatory response, or discovery — run three mandatory compliance checks: (1) PII scan to identify personal data that must be redacted, (2) legal hold check to identify items that must be withheld or logged, and (3) privilege check to identify potentially privileged communications requiring a privilege log. Produces a structured clearance report with PASS/FAIL status per check and specific action items for every flagged record.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Define the Export Set

Extract from the user's input the criteria that define the production set:

| Parameter | How to identify | Notes |
|---|---|---|
| Topic / matter | What the export covers | Required |
| Custodians | Whose records are included | Required |
| Date range | Date scope of the export | Required |
| Archive(s) | Which archives are in scope | Required |

If the export set cannot be clearly defined, stop and ask:
> *"Please describe the export set — what topic, custodians, date range, and archives are included in this production?"*

---

## Step 2 — Build Three Parallel Check Queries

### Check 1 — PII Queries (6 queries in parallel):
```
(ENTIREMESSAGE:"social security" OR ENTIREMESSAGE:SSN) AND <export filters>
(ENTIREMESSAGE:"credit card" OR ENTIREMESSAGE:"card number" OR ENTIREMESSAGE:CVV) AND <export filters>
(ENTIREMESSAGE:"date of birth" OR ENTIREMESSAGE:DOB) AND <export filters>
(ENTIREMESSAGE:"home address" OR ENTIREMESSAGE:"personal address") AND <export filters>
(ENTIREMESSAGE:"account number" OR ENTIREMESSAGE:"routing number" OR ENTIREMESSAGE:IBAN) AND <export filters>
(ENTIREMESSAGE:"medical record" OR ENTIREMESSAGE:diagnosis OR ENTIREMESSAGE:prescription) AND <export filters>
```

### Check 2 — Legal Hold Query:
```
CLASSIFICATION.TAGS:hold AND <export filters>
```
Also check `isLegalHold: true` in all results across all checks.

### Check 3 — Privilege Queries (2 queries):
```
CLASSIFICATION.TAGS:privilege AND <export filters>
(ENTIREMESSAGE:"attorney-client" OR ENTIREMESSAGE:"work product" OR ENTIREMESSAGE:privileged OR ENTIREMESSAGE:"under seal") AND <export filters>
```

**Export filters** = custodian + date range from Step 1:
```
FROMORTO:"custodian@domain.com" AND MAILDATE:[YYYYMMDD TO YYYYMMDD]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute All Checks in Parallel

Call all applicable search tools **in parallel** for all check queries simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per query.

---

## Step 4 — Classify Each Flagged Item

For each result across all three checks:

| Check | Classification | Action |
|---|---|---|
| PII — High confidence (value in snippet) | Critical | Redact before export |
| PII — Medium confidence (term only) | Review | Human review before export |
| Legal Hold | Must withhold | Withhold from production + log |
| Privilege — confirmed | Must withhold | Withhold + generate privilege log entry |
| Privilege — possible | Review | Attorney review before export |

For all **Critical** items (PII confirmed, legal hold confirmed, privilege confirmed), call the preview tool using **IDs from this session only** to confirm before recommending action.

---

## Step 5 — Produce the Pre-Export Compliance Report

```
## Arctera Pre-Export Compliance Report

**Export set description:** [topic / matter]
**Custodians:** [list]
**Date range:** [range]
**Archives in scope:** [Email / Files / Collab / All]
**Checks performed:** PII Scan, Legal Hold, Privilege
**Report date:** [today's date — YYYY-MM-DD]

---

### Compliance Check Summary

| Check | Result | Critical | Review | Action Required |
|---|---|---|---|---|
| ☑ PII Scan | ⚠ FAIL / ✓ PASS | N | N | Redact [N] items |
| ☑ Legal Hold | ⚠ FAIL / ✓ PASS | N | — | Withhold [N] items |
| ☑ Privilege | ⚠ FAIL / ✓ PASS | N | N | Withhold [N] / Review [N] |

**Overall export status:** ⚠ NOT CLEARED — resolve [N] items before export
*(or: ✓ CLEARED — no blocking items found)*

---

### Critical Items — Must Resolve Before Export

| # | Check | Archive | Date | Sender | Subject / Filename | Issue | Required Action |
|---|-------|---------|------|--------|--------------------|-------|----------------|
| 1 | PII | Email | ... | ... | ... | SSN: `XXX-XX-####` | Redact |
| 2 | Hold | File | ... | ... | ... | Legal hold active | Withhold + log |
| 3 | Privilege | Email | ... | ... | ... | Attorney-client | Withhold + privilege log |

---

### Review Items — Human Decision Required

| # | Check | Archive | Date | Subject / Filename | Issue | Recommended Action |
|---|-------|---------|------|--------------------|-------|-------------------|

---

### Export Clearance Checklist

- [ ] Redact [N] PII Critical items — then re-run PII check to confirm clearance
- [ ] Withhold [N] legal hold items — log each as withheld in the production log
- [ ] Generate privilege log entries for [N] privilege-confirmed items
- [ ] Attorney review of [N] privilege review items before final decision
- [ ] Human review of [N] PII review items
- [ ] Re-run `/arctera-pre-export` with the same scope after remediation to confirm full clearance
- [ ] Document export date, custodians, withheld count, and redaction count in the production log
```

---

## Behaviour Rules

- Run all three checks — PII, Legal Hold, Privilege — **in parallel** across all applicable archives.
- Run all PII sub-queries and all Privilege sub-queries **in parallel**.
- Only use get tools with IDs from the **current session**.
- **Never reproduce raw PII values** — use masked format (SSN: `XXX-XX-####`, card: `XXXX-XXXX-XXXX-####`).
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- Legal hold status is only confirmed when `isLegalHold: true` is in the returned result or `CLASSIFICATION.TAGS:hold` is confirmed.
- Do not call more than 10 get tools without checking with the user.
- If all three checks pass with zero items, state: *"✓ CLEARED — No PII, legal hold, or privilege items detected in the defined export set."*
