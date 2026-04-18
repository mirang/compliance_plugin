---
name: arctera-insider-risk
description: Detects insider trading signals and MNPI-related communications across all three Arctera archives. Looks for explicit MNPI references, trading restrictions, blackout period discussions, pre-announcement material event communications, and suspicious securities trading language.
type: skill
---

# Skill: Arctera Insider Trading & MNPI Signal Detection

## Purpose
Identify communications that may evidence insider trading activity or the misuse of material non-public information (MNPI). Scans for explicit MNPI references, trading blackout violations, restricted list discussions, and insider communications about material corporate events prior to public announcement. Produces a risk-tiered report distinguishing confirmed MNPI references from contextual risk signals. Used in regulatory response, internal investigations, and proactive conduct risk monitoring.

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_email` | Emails | `get_email` | `mailID` |
| `search_file` | Files / Documents | `get_file` | `fileId`, `batchId`, `messageId` |
| `search_message` | Collaboration / Chat | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |

---

## Step 1 — Parse Insider Risk Parameters

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Ticker symbols | Stock ticker patterns (e.g. ENE, MSFT) | Scopes securities-specific queries |
| Custodians | Email addresses or names | Optional |
| Date range | Critical — especially pre-announcement periods | Strongly recommended |
| Archive scope | "emails", "files", "chat", "all" | Default: all three |
| Material events | Merger, acquisition, earnings, restructuring | Optional context |

If no archive keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for insider trading signals?"*

---

## Step 2 — Build MNPI and Insider Risk Queries

```
ENTIREMESSAGE:"material non-public" OR ENTIREMESSAGE:MNPI OR ENTIREMESSAGE:"inside information"
ENTIREMESSAGE:"do not trade" OR ENTIREMESSAGE:"blackout period" OR ENTIREMESSAGE:"trading window"
ENTIREMESSAGE:"restricted list" OR ENTIREMESSAGE:"watch list" OR ENTIREMESSAGE:"grey list"
ENTIREMESSAGE:"non-public" OR ENTIREMESSAGE:"not yet public" OR ENTIREMESSAGE:"before announcement"
ENTIREMESSAGE:"pre-announcement" OR ENTIREMESSAGE:"before we announce" OR ENTIREMESSAGE:"embargoed"
ENTIREMESSAGE:merger OR ENTIREMESSAGE:acquisition OR ENTIREMESSAGE:"going private" [AND MAILDATE: pre-announcement range]
ENTIREMESSAGE:earnings OR ENTIREMESSAGE:guidance OR ENTIREMESSAGE:"quarterly results" [AND MAILDATE: pre-announcement range]
```

If ticker symbols are provided, add:
```
ENTIREMESSAGE:"<TICKER>" AND (ENTIREMESSAGE:buy OR ENTIREMESSAGE:sell OR ENTIREMESSAGE:trade OR ENTIREMESSAGE:position OR ENTIREMESSAGE:short)
```

Append custodian (`FROMORTO:"email"`) and date (`MAILDATE:[YYYYMMDD TO YYYYMMDD]`) filters with AND.

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

**Insider Risk Signal Type:**
| Type | Criteria |
|---|---|
| Explicit MNPI reference | Direct mention of "material non-public information" or MNPI with trading context |
| Trading restriction | Blackout period, restricted list, or "do not trade" instruction |
| Pre-announcement material | Discussion of merger, earnings, or material event before public disclosure |
| Pattern indicator | Insider discussing specific securities near a material corporate event |
| Institutional awareness | Evidence a party knew of material information before it was public |

**Risk Level:**
| Level | Criteria |
|---|---|
| Critical | Explicit MNPI reference + trading language in same communication |
| High | Pre-announcement discussion of material event by an insider |
| Medium | Trading restriction or blackout reference in a context suggesting non-compliance |
| Low | General securities or market reference with no specific insider context |

For **Critical** and **High** items, call the preview tool using **IDs from this session only**:
- Email → `get_email(mailID: <mailID>)`
- File → `get_file(fileId: <fileId>, batchId: <batchId>, messageId: <messageId>)`
- Collab → `get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

---

## Step 5 — Produce the Insider Risk Report

```
## Arctera Insider Trading & MNPI Risk Report

**Tickers in scope:** [list or "none specified"]
**Custodians in scope:** [list or "all"]
**Date range:** [range — especially pre-announcement period]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Total records found:** [N] ([n] emails, [n] files, [n] collab)
**Critical:** [N] | **High:** [N] | **Medium:** [N] | **Low:** [N]

---

### Critical Items — Explicit MNPI + Trading Language

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-------------|

---

### High-Risk Items — Pre-Announcement Material Discussion

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|---------|

---

### Medium Items — Trading Restriction or Blackout Context

| # | Archive | Date | Sender / Author | Subject | Signal Type | Excerpt |
|---|---------|------|-----------------|---------|-------------|---------|

---

### Insider Risk Narrative

[2–3 paragraphs: nature of MNPI exposure, key parties, proximity to material events,
whether restrictions were communicated and appear to have been violated, recommended actions]

---

### Recommended Next Steps

- [ ] Escalate Critical items to General Counsel, Compliance Officer, and outside counsel immediately
- [ ] Preserve all flagged items — run `/arctera-legal-hold`
- [ ] Run `/arctera-custodian` on parties identified in Critical items
- [ ] Run `/arctera-timeline` to map communications against corporate event announcement dates
- [ ] Conduct trading records review for custodians in Critical items
- [ ] Consider voluntary disclosure assessment with securities outside counsel
```

---

## Behaviour Rules

- Run all applicable search tools **in parallel**.
- Date range is especially important for this scan — prioritise pre-announcement periods.
- Only use get tools with IDs from the **current session**.
- For `get_file` results, summarise `html_content` — do not display raw HTML.
- Do not reproduce specific securities positions, quantities, or prices in the report.
- Do not call more than 10 get tools without checking with the user.
