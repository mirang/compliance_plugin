You are running the **Arctera Channel-Specific Communication Audit** workflow.

User request: $ARGUMENTS

---

## 1 — Identify the channel from $ARGUMENTS

Extract:
- **Channel type** — the specific messaging channel to audit:
  - Bloomberg Chat → `SOURCETYPE:bloomberg`
  - Microsoft Teams → `SOURCETYPE:teams`
  - Slack → `SOURCETYPE:slack`
  - Reuters Messaging → `SOURCETYPE:reuters`
  - Any / all collaboration → omit SOURCETYPE filter
- **Topic / keywords** — what to look for in this channel
- **Custodians** — optional specific parties
- **Date range** — optional scope

If no channel can be identified from $ARGUMENTS, ask:
> *"Which channel would you like to audit? (Bloomberg, Teams, Slack, Reuters, or all collaboration channels)"*

---

## 2 — Build channel-targeted queries

**Primary query with channel filter:**
```
ENTIREMESSAGE:"<topic>" AND SOURCETYPE:<channel> [AND FROMORTO:"custodian@domain.com"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Custodian-only query (all topics in this channel):**
```
FROMORTO:"custodian@domain.com" AND SOURCETYPE:<channel> [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Regulatory / risk signals in this channel:**
```
(ENTIREMESSAGE:confidential OR ENTIREMESSAGE:"do not share" OR ENTIREMESSAGE:delete) AND SOURCETYPE:<channel> [AND FROMORTO:...] [AND MAILDATE:...]
```

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search the collaboration archive

Use `search_message` with:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — for timeline view)
- `pageNum: 1`

Paginate while `has_more` is true, up to 100 results.

---

## 4 — Triage and preview

From results, identify:
1. Most active participants in this channel
2. Topics discussed
3. Any risk signals: "delete this", "don't put this in email", "off the record"
4. External parties in the conversation
5. Items on legal hold

For the top 10 most significant items, call `get_message(chatRoomId, collabMessageId, chatRoomName, matterId, collabDate)`.
Only use IDs from this session. Present results as a chat timeline ordered by date ascending.

---

## 5 — Deliver the Channel Audit Report

```
## Arctera Channel Audit — [Channel Type]

**Channel:** [Bloomberg / Teams / Slack / Reuters / All Collaboration]
**Topic filter:** [keyword or "all topics"]
**Custodians in scope:** [list or "all"]
**Date range:** [range or "all dates"]
**Total messages found:** [N]
**Chat rooms / conversations:** [N distinct rooms]

---

### Chat Room / Conversation Summary

| Room Name | Participants | Message Count | Date Range | Topic(s) | Risk Flags |
|-----------|-------------|---------------|------------|----------|------------|

---

### Risk-Flagged Messages

| # | Date | Chat Room | Sender | Message Excerpt | Risk Signal |
|---|------|-----------|--------|-----------------|-------------|

*(Common risk signals: "delete this", "off the record", "don't email this", "call me")*

---

### Channel Activity Timeline

*(Top messages ordered chronologically with key excerpts)*

---

### Participants

| Participant | Message Count | External? | First Activity | Last Activity |
|-------------|---------------|-----------|----------------|---------------|

---

### Recommended Next Steps

- [ ] Full review of room "[room name]" — [N] messages on [topic]
- [ ] Run `/arctera-custodian <name>` for full cross-archive pull on most active participant
- [ ] Run `/arctera-cross-archive` to compare this channel's activity with email and files on same topic
- [ ] Investigate risk signal in item #N — "[excerpt]" — on [date]
```
