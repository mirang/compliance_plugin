---
name: arctera-custodian
description: Pulls all archived communications to/from a specific custodian across email, files, and collaboration. Produces a custodian profile report covering activity summary, external contacts, and notable items — core to any eDiscovery collection.
type: skill
---

# Skill: Arctera Custodian Communication Pull

## Purpose
Pull a complete picture of all archived communications involving a specific custodian across email, files, and collaboration messages. This is the foundational step for eDiscovery collection, deposition preparation, custodian interviews, and investigation scoping. The skill identifies the custodian's communication patterns, key counterparties, external contacts, and items already on legal hold.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Identify the Custodian

Extract from the user's input:

| Parameter | How to identify | Required? |
|---|---|---|
| Email address | Any `name@domain.com` pattern | Primary identifier |
| Full name | Text that looks like a person's name | Used for file/chat searches |
| Date range | "from … to", year references | Optional — recommend always |
| Topic filter | Additional keywords to scope the pull | Optional |

If neither an email address nor a name can be identified, stop and ask:
> *"Please provide the custodian's email address or full name."*

---

## Step 2 — Build Queries per Archive

### Email (sent or received)
```
FROMORTO:"custodian@domain.com" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]] [AND ENTIREMESSAGE:"keyword"]
```

### Files (authored or involved)
```
ENTIREMESSAGE:"First Last" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```
Also:
```
SENDER:"custodian@domain.com" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

### Collaboration / Chat
```
FROMORTO:"custodian@domain.com" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```
Also try name:
```
ENTIREMESSAGE:"First Last" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID
- For domain-level sender: use `SENDER:domain*` — NEVER `SENDER:*@domain.com`

---

## Step 3 — Execute Searches

Call `search_email`, `search_file`, and `search_message` **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 total results per archive.

---

## Step 4 — Triage and Prioritise

From all results, identify the most significant items:

1. **Legal hold items**: `isLegalHold: true` — always flag and list
2. **External communications**: any item to/from a non-corporate domain (personal email, competitor, regulator, law firm)
3. **Executive involvement**: items involving C-suite names or board members
4. **Large attachments**: `ATTCOUNT > 0` or high attachment size
5. **Earliest and latest items per archive**: establish the custodian's communication window

Select the **top 10 most significant** items for full preview using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Custodian Profile Report

```
## Arctera Custodian Report — [Custodian Name] <[email]>

**Custodian:** [Full name and email address]
**Date range searched:** [range or "all available dates"]
**Topic filter:** [keyword or "none — full pull"]
**Archives searched:** Email, Files, Collaboration

**Total communications found:** [N] ([n] emails, [n] files, [n] collab)
**Reviewed (snippet):** [N] | **Previewed in full:** [N]
**Items on legal hold:** [N]
**External communications:** [N] (to/from non-corporate domains)

---

### Communication Activity Summary

| Archive | Count | Earliest | Latest | Top Counterparties |
|---------|-------|----------|--------|--------------------|
| Email | N | date | date | name1, name2 |
| Files | N | date | date | name1, name2 |
| Collab | N | date | date | name1, name2 |

---

### Notable Items

| # | Archive | Date | With / Subject / Filename | Key Excerpt | Flag |
|---|---------|------|--------------------------|-------------|------|
| 1 | Email | ... | ... | "..." | Legal Hold |
| 2 | File  | ... | ... | "..." | External |

---

### External Contacts

| External Party / Domain | Archive | Communication Count | First Contact | Last Contact |
|------------------------|---------|---------------------|---------------|--------------|

---

### Legal Hold Coverage

| Hold Status | Count | Note |
|---|---|---|
| Already on hold | N | Items with `isLegalHold: true` |
| Not held — recommend review | N | Items matching matter criteria |

---

### Recommended Next Steps

- [ ] Run `/arctera-between <custodian> and <top counterparty>` to investigate key relationships
- [ ] Run `/arctera-legal-hold` to confirm and expand hold coverage for this custodian
- [ ] Run `/arctera-audit-pii` if this custodian's records are being produced
- [ ] Escalate external communications to [domain] to legal counsel for review
- [ ] Review full content of item #N — [reason]
```

---

## Behaviour Rules

- Search all three archives **in parallel** using both email and name-based queries.
- Only use `get_email` / `get_file` / `get_message` with IDs from the **current session**.
- Do not fabricate content — only summarise returned snippets or full previews.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- For `get_message` results, present as a chat timeline ordered by `date` ascending.
- If zero results are found in an archive, say so explicitly — do not omit it from the report.
- Flag every item to/from a personal email domain (gmail, yahoo, hotmail, icloud, outlook.com personal) as external.
