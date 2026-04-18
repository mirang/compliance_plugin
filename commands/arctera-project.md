You are running the **Arctera Project & Deal Communication Pull** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Project / deal name** — the named project, deal, or initiative (e.g. "Project Raptor", "LJM Partnership", "Operation Phoenix")
- **Date range** — optional scope (e.g. "1999 to 2001")
- **Key custodians** — optional parties to focus on

If no project name can be identified, stop and ask:
> *"What is the name of the project or deal you want to pull communications for?"*

---

## 2 — Build project queries

Run both a quoted phrase and a subject-targeted query:

```
ENTIREMESSAGE:"<Project Name>"
SUBJECT:"<Project Name>"
```

Also run abbreviated or code name variants if identifiable:
```
ENTIREMESSAGE:"<abbreviation>"
```

Append custodian and date filters with AND where provided.

**Wildcard rule:** Only suffix wildcard `term*` (min 3 chars). NEVER `*term` or `*term*`.

---

## 3 — Search all three archives in parallel

Call `search_email`, `search_file`, and `search_message` simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `sortAsc: true` (ascending — chronological project history)
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 100 results per archive.

---

## 4 — Triage and preview

From combined results, identify:
1. Project inception communications (earliest items)
2. Key decision points (items referencing approvals, agreements, milestones)
3. External parties involved (non-corporate domains)
4. Files and documents (contracts, models, presentations)
5. Items on legal hold

Preview the top 10 most significant items across all archives using get tools.
Only use IDs from this session.

---

## 5 — Deliver the Project Communication Report

```
## Arctera Project Report — [Project Name]

**Project:** [name]
**Date range:** [range or "all available dates"]
**Archives searched:** Email, Files, Collaboration
**Total communications found:** [N] ([n] emails, [n] files, [n] collab)
**Reviewed:** [N] | **Previewed in full:** [N]

---

### Project Timeline

| # | Date | Archive | Sender / Author | Subject / Filename / Room | Key Excerpt | Milestone? |
|---|------|---------|-----------------|--------------------------|-------------|------------|

---

### Key Participants

| Party | Archive(s) | Role | Communication Count | First Activity | Last Activity |
|-------|------------|------|---------------------|----------------|---------------|

---

### External Parties

| External Party / Domain | Archive | Count | First Contact |
|------------------------|---------|-------|---------------|

---

### Documents & Files

| # | Filename | Date | Author | Key Content |
|---|----------|------|--------|-------------|

---

### Project Narrative

[2–3 paragraphs: project origin, key milestones, decision points, parties involved, and any risk signals]

---

### Recommended Next Steps

- [ ] Run `/arctera-timeline` for a full chronological view of the project
- [ ] Run `/arctera-between <party A> and <party B>` for key relationship pairs
- [ ] Run `/arctera-legal-hold` if this project is subject to litigation or investigation
- [ ] Run `/arctera-audit-pii` before exporting project communications
```
