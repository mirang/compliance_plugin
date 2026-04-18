You are running the **Arctera Insider Trading & MNPI Signal Detection** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Ticker symbols** — optional specific securities (e.g. "ENE", "ENRNQ")
- **Custodians** — optional specific parties to investigate
- **Date range** — critical for pre-announcement period analysis (e.g. "2001-09 to 2001-12")
- **Archive scope** — default: all three archives

If no scope keyword is present, ask:
> *"Would you like to scan emails, files, collaboration messages, or all three for insider trading signals?"*

---

## 2 — Build MNPI and insider risk queries

```
ENTIREMESSAGE:"material non-public" OR ENTIREMESSAGE:MNPI OR ENTIREMESSAGE:"inside information"
ENTIREMESSAGE:"do not trade" OR ENTIREMESSAGE:"blackout period" OR ENTIREMESSAGE:"trading window"
ENTIREMESSAGE:"restricted list" OR ENTIREMESSAGE:"watch list" OR ENTIREMESSAGE:"grey list"
ENTIREMESSAGE:"non-public" OR ENTIREMESSAGE:"not yet public" OR ENTIREMESSAGE:"before announcement"
ENTIREMESSAGE:merger OR ENTIREMESSAGE:acquisition OR ENTIREMESSAGE:"going private" [filtered by date near announcement]
ENTIREMESSAGE:"earnings" OR ENTIREMESSAGE:"guidance" OR ENTIREMESSAGE:"results" [filtered by pre-announcement date]
```

If ticker symbols are provided:
```
ENTIREMESSAGE:"<TICKER>" AND (ENTIREMESSAGE:"buy" OR ENTIREMESSAGE:"sell" OR ENTIREMESSAGE:"trade" OR ENTIREMESSAGE:"position")
```

Append custodian and date filters with AND where provided.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search applicable archives in parallel

Call search tools simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 results per query.

---

## 4 — Classify each result

**Signal Type:**
- Explicit MNPI reference: direct mention of material non-public information
- Trading restriction: blackout period, restricted list, do not trade instruction
- Pre-announcement discussion: material event discussed before public disclosure
- Pattern indicator: discussion of specific securities by an insider near a material event
- Institutional awareness: evidence a party was aware of material information

**Risk Level:**
- Critical: explicit MNPI reference with trading language
- High: pre-announcement discussion of material event by insider
- Medium: trading restriction or blackout reference in relevant context
- Low: general market or securities reference without insider context

For Critical and High items, call `get_email`, `get_file`, or `get_message` to confirm.
Only use IDs from this session.

---

## 5 — Deliver the Insider Risk Report

```
## Arctera Insider Trading & MNPI Risk Report

**Tickers in scope:** [list or "none specified"]
**Custodians in scope:** [list or "all"]
**Date range:** [range — especially pre-announcement period]
**Archives searched:** [Email / Files / Collab / All]
**Queries run:** [N]
**Critical items:** [N] | **High:** [N] | **Medium:** [N]

---

### Critical Items — Explicit MNPI Reference with Trading Language

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Key Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|-------------|

---

### High-Risk Items — Pre-Announcement Material Discussion

| # | Archive | Date | Sender / Author | Subject / Filename / Room | Signal Type | Excerpt |
|---|---------|------|-----------------|--------------------------|-------------|---------|

---

### Risk Narrative

[2–3 paragraphs: nature of insider risk, key parties, proximity to material events, recommended escalation]

---

### Recommended Next Steps

- [ ] Escalate Critical items to General Counsel and Compliance Officer immediately
- [ ] Preserve all flagged items — run `/arctera-legal-hold`
- [ ] Run `/arctera-custodian` on parties identified in Critical items
- [ ] Run `/arctera-timeline` to map communications against announcement dates
- [ ] Notify outside counsel and consider voluntary disclosure if warranted
```
