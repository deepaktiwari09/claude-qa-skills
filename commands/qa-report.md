---
description: Generate a QA knowledge base report from accumulated session data. Creates an email-ready .md report for stakeholders.
argument-hint: [what to report on — e.g. "full report", "UI flows only", "DB costs", "improvements backlog"]
allowed-tools: Bash, Read, Write, Glob, Grep, Edit
---

# QA Knowledge Report Generator

You are generating a structured report from the project's QA knowledge base. The report will be saved as `.md` and formatted for copy-paste into email (no tables — use bullet lists and headers instead).

## Context

- Project: !`basename $(pwd)`
- Branch: !`git branch --show-current 2>/dev/null`
- Date: !`date +%Y-%m-%d`

## User Request

$ARGUMENTS

## Step 1: Find the Knowledge Base

Look for accumulated QA data in this order:

```
e2e/knowledge/           → primary location
docs/qa/                 → alternate location
```

Check what's available:
- `e2e/knowledge/flows/` → flow maps
- `e2e/knowledge/sessions/` → session reports
- `e2e/knowledge/pain-points.md` → friction tracker
- `e2e/knowledge/improvements.md` → improvement backlog
- `e2e/knowledge/metrics.md` → trend data
- `e2e/knowledge/db/` → DB business logic KB

If no knowledge base exists, tell the user: "No QA knowledge base found. Run `/qa` on some flows first to build up data, then come back here to generate a report."

## Step 2: Determine Report Type

Based on user's request, pick one:

### Full QA Report
Covers everything — UI flows, DB analysis, pain points, improvements, costs. Best for monthly stakeholder updates or sprint retrospectives.

### UI Flows Report
Focused on user flow health — step counts, friction, improvement candidates. Best for PM/UX team.

### DB Health Report
Focused on schema, business rules, query costs, stability risks. Best for backend/DevOps team.

### Improvements Report
Focused on the backlog — quick wins, medium effort, strategic items. Best for sprint planning.

### Pain Points Report
Focused on accumulated friction — critical, major, minor. Best for bug triage meetings.

### Custom Report
User specifies exactly what they want. Extract relevant sections.

## Step 3: Generate the Report

**CRITICAL: No markdown tables in the report.** Tables break when copy-pasted into email clients. Use bullet lists, numbered lists, and headers instead.

### Email-Friendly Formatting Rules

Instead of:
```
| Flow | Steps | Time |
|------|-------|------|
| Login | 3 | 12s |
```

Use:
```
**Login Flow**
- Steps: 3
- Avg time: 12s
- Health: Good
```

Or for lists:
```
1. **Login** — 3 steps, 12s, Good
2. **Registration** — 5 steps, 45s, Needs work
3. **Checkout** — 4 steps, 30s, Review
```

## Step 4: Report Templates

### Full QA Report

```markdown
# QA Knowledge Report — [Project Name]

**Generated:** [Date]
**Period:** [First session date] to [Last session date]
**Total QA Sessions:** [N]
**Report by:** [User name or "Claude Code (automated)"]

---

## Executive Summary

[2-3 sentences: overall health, biggest risks, top recommendation]

---

## UI Flow Health

### Flow Overview

[For each flow in flows/ directory:]

**[Flow Name]**
- Entry: [URL]
- Steps: [N] (target: [M])
- Avg time: [Xs]
- Error rate: [N%]
- Health: [Good / Review / Needs Work]
- State snapshots: [Available / None]

### Flows Needing Attention

[List flows with health = "Needs Work" or step count > 6]

1. **[Flow Name]** — [N] steps, [reason it needs attention]
   - Suggested fix: [from improvements.md]

---

## Pain Points Summary

### Critical (Blocks User Goal)

[From pain-points.md, critical section]

1. **[Date]** [Description]
   - Impact: [who is affected, how often]
   - Status: [Open / In Progress / Fixed]

### Major (Causes Frustration)

[From pain-points.md, major section]

1. **[Date]** [Description]

### Minor (Annoyance)

[From pain-points.md, minor section — summarize count only]

- [N] minor issues logged (see full list in e2e/knowledge/pain-points.md)

---

## Database Health

### Schema Overview

- Tables: [N]
- Total rows: [N]
- Database size: [X MB/GB]

### Business Rules Audit

**Rules enforced at all layers (DB + Backend + Frontend):** [N]
**Rules missing DB enforcement:** [N] — these are risk areas

[List rules missing DB enforcement:]

1. **[Rule]** — Currently only in [Backend/Frontend]. Risk: [description]

### Query Cost Analysis

**Top expensive queries:**

1. **[Query pattern]** — [N] calls/day, avg [X]ms
   - Fix: [recommendation]
   - Estimated savings: [description]

2. **[Query pattern]** — [N] calls/day, avg [X]ms
   - Fix: [recommendation]

### Stability Risks

**Critical:**
1. [Risk description + fix recommendation]

**Warning:**
1. [Risk description]

---

## Improvement Backlog

### Quick Wins (Low Effort, High Impact)

1. [Improvement] — Impact: [High/Medium], Effort: [Low]
2. [Improvement]

### Next Sprint Candidates (Medium Effort)

1. [Improvement] — Impact: [High/Medium], Effort: [Medium]

### Strategic (Needs Design/PM Review)

1. [Improvement] — Impact: [High], Effort: [High]

---

## QA Session History

**Recent sessions:**

1. **[Date] — [Flow tested]**
   - Result: [N] passed, [N] failed
   - Key finding: [one-liner]

2. **[Date] — [Flow tested]**
   - Result: [N] passed, [N] failed
   - Key finding: [one-liner]

[Show last 5-10 sessions]

---

## Recommendations

### Immediate Actions (This Week)

1. [Action item with clear owner suggestion]

### Short Term (This Sprint)

1. [Action item]

### Long Term (Next Quarter)

1. [Action item]

---

## Metrics Trend

**Flow health over time:**

- [Month]: [N] flows healthy, [N] need work
- [Month]: [N] flows healthy, [N] need work

**Issues found vs fixed:**

- [Month]: [N] found, [N] fixed, [N] open
- [Month]: [N] found, [N] fixed, [N] open

---

*This report was generated from the project's QA knowledge base at e2e/knowledge/.*
*To update the knowledge base, run /qa on any feature or flow.*
```

### UI Flows Report (Shorter)

```markdown
# UI Flow Health Report — [Project Name]

**Generated:** [Date]
**Flows analyzed:** [N]

---

## Flow Summary

[For each flow:]

**[Flow Name]**
- Steps: [N] (target: [M])
- Avg time: [Xs]
- Health: [Good / Review / Needs Work]
- Top friction: [one-liner from pain points]
- Improvement: [top suggestion]

---

## Flows Ranked by Health

**Needs Immediate Attention:**
1. [Flow] — [reason]

**Review Soon:**
1. [Flow] — [reason]

**Healthy:**
1. [Flow]

---

## Top 5 UX Improvements

1. [Improvement] — affects [flow], impact [High/Medium]
2. ...

---

*Run /qa on flagged flows to gather fresh data.*
```

### DB Health Report (Shorter)

```markdown
# Database Health Report — [Project Name]

**Generated:** [Date]
**Database:** [type + version]
**Total tables:** [N]
**Total rows:** [N]

---

## Business Rules at Risk

[Rules with gaps in enforcement:]

1. **[Rule]** — DB: [Yes/No], Backend: [Yes/No], Frontend: [Yes/No]
   - Risk: [description]
   - Fix: [recommendation]

---

## Top Expensive Queries

1. **[Pattern]** — [calls/day] x [avg ms] = [daily CPU seconds]
   - Service: [file:method]
   - Fix: [index/cache/restructure]

---

## Stability Risks

1. **[Risk]** — Severity: [Critical/Warning]
   - Fix: [recommendation]

---

## Cost Optimization Summary

- Current estimated daily CPU: [Xs]
- After optimizations: [Ys] ([Z%] reduction)
- Infrastructure recommendation: [sizing advice]

---

*Run /qa with DB testing to refresh this data.*
```

## Step 5: Save and Offer to Send

1. Save the report to `e2e/reports/qa-report-[DATE].md`
2. If `e2e/reports/` doesn't exist, create it
3. Show the user the full report content
4. Ask: "Report saved. Want me to:"
   - "Copy-friendly version is above — paste into your email client"
   - "Create a Jira ticket with this report attached?"
   - "Generate a shorter executive summary for leadership?"

## Rules

1. **No markdown tables** — they break in email. Use bullet lists, numbered lists, headers
2. **Only report on data that exists** — skip empty sections, don't fabricate metrics
3. **Always show source** — mention which knowledge base files the data came from
4. **Date everything** — timestamps on reports, sessions, pain points
5. **Actionable language** — every finding should have a recommendation
6. **Save to e2e/reports/** — build a history of reports over time
7. **Keep it scannable** — busy stakeholders read headers first, details second
