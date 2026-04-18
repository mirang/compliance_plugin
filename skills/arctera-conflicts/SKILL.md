---
name: arctera-conflicts
description: Conflicts of interest detection across all three Arctera archives. Identifies undisclosed financial interests, self-dealing, recusal failures, outside business activities, and dual-role situations — producing a risk-tiered conflicts report.
type: skill
---

# Skill: Arctera Conflicts of Interest Detection

## Purpose
Detect communications and documents evidencing conflicts of interest across the compliance archive. Identifies undisclosed financial interests in transactions, self-dealing, participation where recusal was required, outside business activities, and COI disclosure process discussions. Useful for investment banking Chinese wall compliance, public company governance, regulatory examination response, and internal investigations.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Conflicts Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Archive scope | "emails", "files", "chat", "all" | Default: all three |
| Date range | Year or date range | Optional |
| Custodians | Email addresses or names | Optional |
| Transaction / matter | Named deal or matter | Optional — scopes the search |

If no archive keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for conflicts of interest?"*

---

## Step 2 — Build Conflicts Signal Queries

```
ENTIREMESSAGE:"conflict of interest" OR ENTIREMESSAGE:"conflicts of interest"
ENTIREMESSAGE:recuse OR ENTIREMESSAGE:recusal OR ENTIREMESSAGE:"step aside" OR ENTIREMESSAGE:"abstain"
ENTIREMESSAGE:"undisclosed" AND (ENTIREMESSAGE:interest OR ENTIREMESSAGE:relationship OR ENTIREMESSAGE:financial)
ENTIREMESSAGE:"related party" OR ENTIREMESSAGE:"arm's length" OR ENTIREMESSAGE:"arm-length transaction"
ENTIREMESSAGE:"self-dealing" OR ENTIREMESSAGE:"dual role" OR ENTIREMESSAGE:"wearing two hats"
ENTIREMESSAGE:"outside business" OR ENTIREMESSAGE:"outside interest" OR ENTIREMESSAGE:"secondary employment"
ENTIREMESSAGE:"disclosure form" OR ENTIREMESSAGE:"COI form" OR ENTIREMESSAGE:"conflict disclosure"
ENTIREMESSAGE:"personal benefit" OR ENTIREMESSAGE:"personal gain" OR ENTIREMESSAGE:"financial benefit"
```

Append custodian (`FROMORTO:"email"`) and date (`MAILDATE:[YYYYMMDD TO YYYYMMDD]`) filters with AND where provided.

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call all applicable search tools **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for `search_email`; `pageNum: 1` for `search_file` and `search_message`

Paginate while `has_more` is true, up to 100 results per query.

---

## Step 4 — Classify Each Result

**Conflict Type:**
| Type | Criteria |
|---|---|
| Undisclosed financial interest | Personal financial stake in a transaction not disclosed |
| Self-dealing | Party appears on both sides of a transaction or arrangement |
| Recusal failure | Evidence of participation in a matter where recusal was required |
| Outside business activity | Undisclosed external business activity or employment |
| COI disclosure process | Reference to COI disclosure form or process (may be compliant — contextualise) |
| Dual role concern | Party acting in two conflicting capacities simultaneously |

**Risk Level:**
| Level | Criteria |
|---|---|
| Critical | Undisclosed financial conflict with material impact on a decision or transaction |
| High | Evidence of participation where recusal was expressly required |
| Medium | Outside business activity reference or COI concern raised without clear resolution |
| Low | General COI policy, disclosure process, or form reference |

For **Critical** and **High** items, call the preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Conflicts Report

```
## Arctera Conflicts of Interest Report

**Transaction / matter in scope:** [name or "all matters"]
**Custodians in scope:** [list or "all"]
**Date range:** [range or "all dates"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Total records found:** [N] ([n] emails, [n] files, [n] collab)
**Critical:** [N] | **High:** [N] | **Medium:** [N] | **Low:** [N]

---

### Critical Items — Undisclosed Conflict or Self-Dealing

| # | Archive | Date | Custodian | Subject / Filename / Room | Conflict Type | Key Excerpt |
|---|---------|------|-----------|--------------------------|---------------|-------------|

---

### High-Risk Items — Recusal Failure or Participation Concern

| # | Archive | Date | Custodian | Subject / Filename / Room | Conflict Type | Excerpt |
|---|---------|------|-----------|--------------------------|---------------|---------|

---

### Medium Items — Outside Activity or Unresolved COI Concern

| # | Archive | Date | Custodian | Subject / Filename / Room | Conflict Type | Excerpt |
|---|---------|------|-----------|--------------------------|---------------|---------|

---

### Conflicts Narrative

[2–3 paragraphs: nature of conflicts identified, parties and transactions involved,
whether disclosure processes appear to have been followed, recommended actions]

---

### Recommended Next Steps

- [ ] Escalate Critical items to General Counsel and Audit Committee immediately
- [ ] Verify COI disclosure records for custodians named in Critical and High-risk items
- [ ] Run `/arctera-between <party A> and <party B>` for parties on both sides of flagged transactions
- [ ] Run `/arctera-matter <transaction name>` for a full matter review
- [ ] Preserve flagged items if litigation risk is elevated — run `/arctera-legal-hold`
```

---

## Behaviour Rules

- Run all applicable search tools **in parallel**.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- Do not call more than 10 get tools without checking with the user.
- COI disclosure process items (Low) should be noted but not escalated unless surrounding context raises concern.
