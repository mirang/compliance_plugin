---
name: arctera-channel-audit
description: Audits a specific communication channel (Bloomberg Chat, Microsoft Teams, Slack, Reuters Messaging) for a topic or person. Uses SOURCETYPE filtering to isolate channel-specific content, identify active participants and chat rooms, and flag risk signals unique to real-time messaging.
type: skill
---

# Skill: Arctera Channel-Specific Communication Audit

## Purpose
Audit a specific collaboration or messaging channel for a named topic or custodian. Real-time messaging channels (Bloomberg, Teams, Slack, Reuters) present unique compliance risks: communications may be informal, unfiltered, and may reference decisions made away from email. This skill surfaces those communications, identifies active participants and chat rooms, and flags risk signals common in real-time messaging (deletion requests, off-record references, evasive language).

---

## Available Tools

| Tool | Archive | Preview Tool | Key ID Fields |
|---|---|---|---|
| `search_message` | Collaboration / Chat (primary) | `get_message` | `chatRoomId`, `messageId`, `matterId`, `date` |
| `search_email` | Email (supplementary for Bloomberg/Reuters) | `get_email` | `mailID` |

---

## Step 1 — Identify the Channel and Scope

Extract from the user's input:

| Parameter | How to identify | Notes |
|---|---|---|
| Channel type | Bloomberg, Teams, Slack, Reuters, "all collab" | Required |
| Topic / keywords | The subject to audit | Recommended |
| Custodians | Email addresses or names | Optional |
| Date range | Year or date range | Optional |

**SOURCETYPE mapping:**
| User says | Lucene filter |
|---|---|
| Bloomberg, Bloomberg Chat | `SOURCETYPE:bloomberg` |
| Teams, Microsoft Teams | `SOURCETYPE:teams` |
| Slack | `SOURCETYPE:slack` |
| Reuters, Reuters Messaging | `SOURCETYPE:reuters` |
| All collab, all chat | Omit SOURCETYPE (search all message types) |

If no channel can be identified, stop and ask:
> *"Which channel would you like to audit? (Bloomberg, Teams, Slack, Reuters, or all collaboration channels)"*

---

## Step 2 — Build Channel-Targeted Queries

**Primary query — topic within channel:**
```
ENTIREMESSAGE:"<topic>" AND SOURCETYPE:<channel> [AND FROMORTO:"custodian@domain.com"] [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Custodian query — all messages in this channel from custodian:**
```
FROMORTO:"custodian@domain.com" AND SOURCETYPE:<channel> [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

**Risk signal query — evasive language in this channel:**
```
(ENTIREMESSAGE:"don't put this in email" OR ENTIREMESSAGE:"delete this" OR ENTIREMESSAGE:"off the record" OR ENTIREMESSAGE:"call me" OR ENTIREMESSAGE:"don't write this down") AND SOURCETYPE:<channel> [AND MAILDATE:...]
```

### Wildcard rules — STRICT
- ONLY suffix wildcard: `term*` (minimum 3 characters before `*`)
- NEVER `*term` — INVALID
- NEVER `*term*` — INVALID

---

## Step 3 — Execute Searches in Parallel

Call `search_message` with all three queries **in parallel**:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — for chat timeline reconstruction)
- `pageNum: 1`

Paginate while `has_more` is true, up to 100 results per query.

For Bloomberg and Reuters channels, also call `search_email` with equivalent queries (these may be captured as email-type records):
- `pageNum: 0`

---

## Step 4 — Triage and Identify Risk Signals

From all results, identify:

1. **Active chat rooms**: which rooms/channels have the most activity on this topic?
2. **Evasive language**: "don't email this", "off the record", "delete", "call me" — flag these as risk signals
3. **External participants**: non-corporate domain participants in chat rooms
4. **Legal hold items**: `isLegalHold: true`
5. **Decision language**: approvals, agreements, or commitments made in chat without corresponding email

For the top 10 most significant items, call the preview tool using **IDs from this session only**:
`get_message(chatRoomId: <chatRoomId>, collabMessageId: <messageId>, chatRoomName: <chatRoomName>, matterId: <matterId>, collabDate: <date>)`

Present results as a chat timeline ordered by date ascending.

---

## Step 5 — Produce the Channel Audit Report

```
## Arctera Channel Audit — [Channel Type]

**Channel:** [Bloomberg / Teams / Slack / Reuters / All Collaboration]
**Topic filter:** [keyword or "all topics"]
**Custodians in scope:** [list or "all"]
**Date range:** [range or "all dates"]

**Total messages found:** [N]
**Distinct chat rooms / conversations:** [N]
**Risk-flagged messages:** [N]
**Items on legal hold:** [N]

---

### Chat Room / Conversation Summary

| Chat Room | Participants | Message Count | Date Range | Primary Topic | Risk Flags |
|-----------|-------------|---------------|------------|---------------|------------|

---

### Risk-Flagged Messages — Evasive or Off-Record Language

| # | Date | Chat Room | Sender | Message Excerpt | Risk Signal |
|---|------|-----------|--------|-----------------|-------------|
*(Signals: "delete this", "off the record", "don't put in email", "call me", "don't write this down")*

---

### Decisions Made in Chat (No Email Equivalent)

| # | Date | Chat Room | Parties | Decision / Commitment | Follow-up in Email? |
|---|------|-----------|---------|----------------------|---------------------|

---

### Channel Activity Timeline

| # | Date | Chat Room | Sender | Message Excerpt | Flag |
|---|------|-----------|--------|-----------------|------|
*(ordered oldest → newest)*

---

### Participants

| Participant | Messages | External? | Active Rooms | First Message | Last Message |
|-------------|----------|-----------|--------------|---------------|--------------|

---

### Recommended Next Steps

- [ ] Full review of room "[room name]" — contains [N] risk-flagged messages
- [ ] Investigate evasive language in item #N — "[excerpt]" — dated [date]
- [ ] Run `/arctera-custodian <name>` for the most active participant for a full cross-archive pull
- [ ] Run `/arctera-cross-archive` to compare this channel's activity with email and files on the same topic
- [ ] Decisions made in chat at [date] — confirm corresponding email approval exists
```

---

## Behaviour Rules

- Always apply the `SOURCETYPE:` filter when the user specifies a channel — this is the defining feature of this skill.
- Run all three queries **in parallel**.
- Sort results ascending by date — chat must be presented as a timeline.
- Only use get tools with IDs from the **current session**.
- For `get_message`, check the `text` field of each message in the returned `collab_messages` list.
- Evasive language signals ("delete this", "off the record") are **always** flagged as High-risk regardless of topic.
- Do not call more than 10 get tools without checking with the user.
