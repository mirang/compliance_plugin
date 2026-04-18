---
name: arctera-external
description: Audits all outbound communications sent from the corporate domain to external recipients. Identifies personal email transmissions, unknown external contacts, attachments sent outside the organisation, and unusual outbound patterns — supporting data leakage detection and outbound communication compliance.
type: skill
---

# Skill: Arctera External Communication Audit

## Purpose
Audit all outbound communications sent from the corporate domain to external recipients — personal email accounts, unknown third parties, competitors, or unapproved external domains. Identifies what left the organisation, who sent it, and whether the transmission presents a data leakage, IP protection, or regulatory disclosure concern. Produces a risk-tiered outbound audit report.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails (primary for outbound) | `get_email` | `mailID` |
| `search_file` | Files / Documents (supplementary) | `get_file` | `fileId`, `batchId`, `messageId` |

---

## Step 1 — Parse External Audit Parameters

Extract from the user's input:

| Parameter | How to identify | Required? |
|---|---|---|
| Internal domain | Corporate domain(s) | Required |
| Date range | Year or date range | Optional |
| Custodian | Specific sender | Optional |
| Content filter | Keyword to scope | Optional |
| Domain allowlist | Approved external domains | Optional |

If no internal domain can be identified, stop and ask:
> *"What is the corporate email domain? (e.g. enron.com) I need this to identify what counts as 'external'."*

---

## Step 2 — Build Outbound Queries

Run all four queries against email in parallel:

**Query 1 — All outbound with sensitive content markers:**
```
SENDER:<internaldomain>* AND (ENTIREMESSAGE:confidential OR ENTIREMESSAGE:proprietary OR ENTIREMESSAGE:restricted OR ENTIREMESSAGE:"internal only") [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Query 2 — Outbound with attachments:**
```
SENDER:<internaldomain>* AND (ATTACHMENTS.EXTENSION:pdf OR ATTACHMENTS.EXTENSION:xlsx OR ATTACHMENTS.EXTENSION:docx OR ATTACHMENTS.EXTENSION:zip) [AND MAILDATE:...]
```

**Query 3 — Outbound to personal email domains (high-risk exfiltration signal):**
```
SENDER:<internaldomain>* AND (FROMORTO:gmail.com OR FROMORTO:yahoo.com OR FROMORTO:hotmail.com OR FROMORTO:icloud.com OR FROMORTO:protonmail.com) [AND MAILDATE:...]
```

**Query 4 — Outbound with legal or privilege markers:**
```
SENDER:<internaldomain>* AND (ENTIREMESSAGE:"attorney-client" OR ENTIREMESSAGE:privileged OR ENTIREMESSAGE:"work product") [AND MAILDATE:...]
```

If a custodian is specified, add `AND SENDER:"custodian@internaldomain.com"` to each query.

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call `search_email` with all four queries and `search_file` with Query 1 **simultaneously**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file`

Paginate while `has_more` is true, up to 100 results per query.

---

## Step 4 — Classify Each Result

**Recipient Category:**
| Category | Criteria |
|---|---|
| Personal email | gmail, yahoo, hotmail, icloud, outlook.com personal, protonmail |
| Unknown external | Non-corporate domain not on the domain allowlist |
| External counsel | Recognised law firm domain |
| Regulator | SEC, FINRA, FCA, DOJ, CFTC, OCC, etc. |
| Approved counterparty | Domain on the user-provided allowlist |

**Risk Level:**
| Level | Criteria |
|---|---|
| High | Sensitive content OR attachment sent to personal email or unknown external |
| Medium | Any attachment or sensitive content sent to known external party |
| Low | Standard business communication to approved external domain |

For **High-risk** items, call `get_email(mailID)` to confirm the transmission — using **IDs from this session only**.

---

## Step 5 — Produce the External Communication Audit Report

```
## Arctera External Communication Audit

**Internal domain(s):** [domain]
**Custodian filter:** [email or "all internal senders"]
**Date range:** [range or "all dates"]
**Content / attachment filter:** [description or "all outbound"]

**Total outbound communications found:** [N]
**High-risk:** [N] | **Medium-risk:** [N] | **Low-risk:** [N]

---

### High-Risk External Communications — Immediate Review Required

| # | Date | Sender | External Recipient | Recipient Type | Content | Attachment? | Key Excerpt |
|---|------|--------|--------------------|----------------|---------|-------------|-------------|

---

### Medium-Risk External Communications — Review Recommended

| # | Date | Sender | External Recipient | Content | Attachment? | Excerpt |
|---|------|--------|--------------------|---------|-------------|---------|

---

### External Communication by Recipient Type

| Recipient Type | Total | High Risk | Medium Risk | Top Senders |
|---|---|---|---|---|
| Personal email | N | N | N | [names] |
| Unknown external | N | N | N | [names] |
| External counsel | N | N | N | [names] |
| Regulator | N | N | N | [names] |
| Approved counterparty | N | N | N | [names] |

---

### Top External Senders

| Sender | Total External | Personal Email Sends | Unknown External Sends | Attachments Sent |
|--------|---------------|---------------------|----------------------|-----------------|

---

### Recommended Next Steps

- [ ] Investigate item #N — [content type] sent to [personal/unknown domain] on [date]
- [ ] Run `/arctera-custodian <sender>` for full cross-archive pull on high-volume external senders
- [ ] Run `/arctera-data-exposure` for a focused sensitive-content leakage scan
- [ ] Confirm item #N was authorised disclosure — external counsel, regulator, or approved counterparty
- [ ] Consider access control review for top external senders
```

---

## Behaviour Rules

- Run all four outbound queries **in parallel** against email; run Query 1 against files simultaneously.
- Only use get tools with IDs from the **current session**.
- For `get_email` results, report attachment names and content summary — do not display raw HTML.
- Flag every transmission to a personal email domain as **High-risk** regardless of content.
- If a domain allowlist is provided, exclude those domains from flagging.
- Do not call more than 10 get tools without checking with the user.
