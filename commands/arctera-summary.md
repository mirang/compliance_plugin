You are running the **Arctera Executive Summary Report** workflow.

User request: $ARGUMENTS

---

## 1 — Parse parameters from $ARGUMENTS

Extract:
- **Matter / topic** — the subject of the summary (e.g. "Enron SEC investigation 2001", "Project LJM")
- **Date range** — optional scope
- **Audience** — optional: who will read this (e.g. "board", "audit committee", "regulators")
- **Archive scope** — default: all three archives

This skill produces a high-level, non-technical executive summary. No raw PII, no Lucene queries in the output, no jargon. Structured for a board-level or senior management audience.

If no topic can be identified, ask:
> *"What matter or topic would you like an executive summary for?"*

---

## 2 — Search all archives in parallel

Build a concise query from the topic:
```
ENTIREMESSAGE:"<topic>" [AND MAILDATE:[YYYYMMDD TO YYYYMMDD]]
```

Call `search_email`, `search_file`, and `search_message` simultaneously:
- `contentDetails: 1` (snippets)
- `pageSize: 20`
- `pageNum: 0` for email; `pageNum: 1` for files and messages

Paginate while `has_more` is true, up to 60 total results per archive (sufficient for a summary).

---

## 3 — Identify the 5 most significant items

From all results, select the 5 most significant items based on:
1. Items with explicit risk language (regulatory, legal, financial)
2. Items involving senior executives or external counsel
3. Items with `isLegalHold: true`
4. Most recent and most historical items (to establish timeline)

Call the preview tool for each. Only use IDs from this session.

---

## 4 — Deliver the Executive Summary Report

Write for a non-legal, executive audience. No technical query language. No raw PII. Plain English.

```
## Executive Summary — [Topic]

**Prepared for:** [audience or "senior management"]
**Date:** [today's date]
**Period covered:** [date range]
**Archives reviewed:** Email, Files, Collaboration

---

### Overview

[1 paragraph: what this summary covers, why it was prepared, and the overall conclusion]

---

### Key Findings

[3–5 bullet points, each one sentence, summarising the most significant findings]

- [Finding 1]
- [Finding 2]
- [Finding 3]

---

### Volume of Communications

| Archive | Records Found | Date Range | Key Parties |
|---------|---------------|------------|-------------|
| Email | N | range | names |
| Files | N | range | names |
| Collaboration | N | range | names |

---

### Risk Indicators

[1–2 paragraphs covering the nature and level of risk identified — written for a non-lawyer]

| Risk Category | Level | Summary |
|---|---|---|
| Legal / Regulatory | High / Medium / Low | [one sentence] |
| Data Privacy | High / Medium / Low | [one sentence] |
| Communications Risk | High / Medium / Low | [one sentence] |

---

### Recommended Actions

[3–5 clear, actionable recommendations in plain English]

1. [Action 1 — what, who, by when]
2. [Action 2]
3. [Action 3]

---

### Notes for Legal Team

[Brief technical note for legal counsel — which commands to run next, what was NOT reviewed]

- Full detailed review: run `/arctera-matter <topic>`
- PII check before export: run `/arctera-audit-pii`
- Legal hold assessment: run `/arctera-legal-hold`
```

**Rules:**
- No raw PII anywhere in this report
- No Lucene syntax in the output
- No archive IDs visible to the reader
- Plain English throughout — assume a non-lawyer audience
