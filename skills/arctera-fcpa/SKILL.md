---
name: arctera-fcpa
description: FCPA and anti-bribery compliance scan across all three Arctera archives. Detects improper payment references, government official interactions, third-party agent arrangements, gifts and entertainment concerns, and internal self-disclosure of potential violations.
type: skill
---

# Skill: Arctera FCPA & Anti-Bribery Compliance Scan

## Purpose
Scan the compliance archive for signals of Foreign Corrupt Practices Act (FCPA) violations and broader anti-bribery risk under the UK Bribery Act and equivalent regimes. Identifies explicit improper payment references, high-risk third-party agent arrangements, excessive gifts and entertainment to government officials, internal self-disclosure, and undisclosed payment schemes. Produces a risk-tiered report for use in regulatory response, M&A due diligence, and proactive anti-corruption compliance.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse FCPA Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Archive scope | "emails", "files", "chat", "all" | Default: all three |
| Date range | Year or date range | Optional |
| Custodians / third parties | Email addresses, agent names | Optional |
| Geography | Country or region | High-risk jurisdiction focus |

If no archive keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for FCPA and anti-bribery signals?"*

---

## Step 2 — Build FCPA Signal Queries

```
ENTIREMESSAGE:"facilitation payment" OR ENTIREMESSAGE:kickback OR ENTIREMESSAGE:bribery OR ENTIREMESSAGE:bribe
ENTIREMESSAGE:"government official" OR ENTIREMESSAGE:"public official" OR ENTIREMESSAGE:"foreign official"
ENTIREMESSAGE:"gifts and entertainment" OR ENTIREMESSAGE:hospitality OR ENTIREMESSAGE:"gift policy"
ENTIREMESSAGE:"third-party agent" OR ENTIREMESSAGE:intermediary OR ENTIREMESSAGE:"representative agreement"
ENTIREMESSAGE:"success fee" OR ENTIREMESSAGE:"finder's fee" OR ENTIREMESSAGE:"commission agreement"
ENTIREMESSAGE:"cash payment" OR ENTIREMESSAGE:"off the books" OR ENTIREMESSAGE:"undisclosed payment"
ENTIREMESSAGE:FCPA OR ENTIREMESSAGE:"UK Bribery Act" OR ENTIREMESSAGE:"anti-corruption" OR ENTIREMESSAGE:"anti-bribery"
ENTIREMESSAGE:"voluntary disclosure" OR ENTIREMESSAGE:"self-report" [in FCPA context]
```

If a specific geography is provided, add:
```
ENTIREMESSAGE:"<country>" AND (ENTIREMESSAGE:official OR ENTIREMESSAGE:payment OR ENTIREMESSAGE:gift OR ENTIREMESSAGE:agent)
```

Append custodian and date filters with AND where provided.

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

**FCPA Signal Type:**
| Type | Criteria |
|---|---|
| Direct bribery | Explicit improper payment to government official |
| Facilitation payment | Payment to expedite routine government action |
| Third-party agent risk | High-commission agent in high-risk jurisdiction with official access |
| Gifts & entertainment | Potentially excessive hospitality to government official |
| Self-disclosure | Internal discussion of potential FCPA violation or voluntary disclosure |
| Anti-corruption policy | Internal policy reference (may be benign — contextualise) |

**Risk Level:**
| Level | Criteria |
|---|---|
| Critical | Explicit improper payment or confirmed self-disclosure of violation |
| High | Third-party agent in high-risk jurisdiction + unusual compensation + official access |
| Medium | Gifts/entertainment to officials without clear policy compliance context |
| Low | General FCPA or anti-bribery policy reference |

For **Critical** and **High** items, call the preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the FCPA Risk Report

```
## Arctera FCPA & Anti-Bribery Risk Report

**Geography focus:** [country/region or "all jurisdictions"]
**Custodians / third parties in scope:** [list or "all"]
**Date range:** [range or "all dates"]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Total records found:** [N] ([n] emails, [n] files, [n] collab)
**Critical:** [N] | **High:** [N] | **Medium:** [N] | **Low:** [N]

---

### Critical Items — Explicit Improper Payment or Self-Disclosure

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Geography | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-----------|-------------|

---

### High-Risk Items — Agent Risk or Hospitality Concerns

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Geography | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-----------|---------|

---

### FCPA Risk Narrative

[2–3 paragraphs: nature of FCPA exposure, jurisdictions involved, third-party relationships,
government official interactions, and recommended escalation path]

---

### Recommended Next Steps

- [ ] Engage outside counsel with FCPA expertise immediately for Critical items
- [ ] Preserve all Critical and High-risk items — run `/arctera-legal-hold`
- [ ] Run `/arctera-between <agent/intermediary> and <internal custodian>` for full relationship view
- [ ] Review third-party agent due diligence files in the archive — run `/arctera-project <agent name>`
- [ ] Consider voluntary disclosure assessment with DOJ/SEC outside counsel
- [ ] Escalate to Board Audit Committee if Critical items are confirmed
```

---

## Behaviour Rules

- Run all applicable search tools **in parallel**.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- Do not reproduce raw payment amounts or financial details — describe the nature of the payment.
- Do not call more than 10 get tools without checking with the user.
- If geography is specified, run the geography-targeted query first before the full sweep.
