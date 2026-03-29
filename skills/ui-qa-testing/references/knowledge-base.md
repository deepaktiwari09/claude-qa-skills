# UI QA Knowledge Base — Reference

Every UI QA session feeds a per-project knowledge base that tracks user flows, friction, and improvement opportunities. This document is the detailed reference for building and maintaining that knowledge base.

## Knowledge Base Location

```
{project}/e2e/knowledge/
├── README.md                      → overview + how to analyze
├── flows/                         → one file per user flow
│   ├── login.flow.md
│   ├── registration.flow.md
│   └── checkout.flow.md
├── sessions/                      → one file per QA session
│   ├── 2026-03-29-login-qa.md
│   └── 2026-03-30-checkout-qa.md
├── pain-points.md                 → accumulated friction by severity
├── improvements.md                → prioritized UX backlog
└── metrics.md                     → cross-session trend data
```

---

## Flow Map — Deep Guide

### What to Capture

For every tested user flow, document:

1. **Entry/exit points** — where does the user start? Where do they end up on success vs failure?
2. **Step count + time** — how many clicks/actions? How long does each take?
3. **Decision points** — where does the user choose between paths? (e.g., social login vs email)
4. **State changes** — what does the app store at each step? (cookies, localStorage, API state)
5. **Error recovery** — what happens when user makes a mistake? Can they go back?
6. **Variants** — different paths through the same flow (new user vs returning, admin vs regular)

### Flow Complexity Score

Rate each flow:

| Score | Steps | Time | Decision Points | Assessment |
|-------|-------|------|-----------------|------------|
| Simple | 1-3 | <15s | 0-1 | Good — leave as is |
| Moderate | 4-6 | 15-60s | 2-3 | Review for shortcuts |
| Complex | 7-10 | 1-3min | 4+ | Needs simplification |
| Excessive | 11+ | 3min+ | 5+ | Split or redesign |

### Flow Comparison Across Sessions

Track how flows change over time:

```markdown
## Flow: Registration — Trend

| Date | Steps | Avg Time | Error Rate | Changes Made |
|------|-------|----------|------------|--------------|
| 2026-03-15 | 6 | 58s | 22% (OTP) | Baseline |
| 2026-03-20 | 5 | 45s | 15% | Merged profile steps |
| 2026-04-01 | 4 | 32s | 8% | Added social login, increased OTP timer |
```

---

## Session Report — What to Track

### Metrics Per Step

| Metric | How to Measure | Why It Matters |
|--------|---------------|----------------|
| Time per step | Timestamps between actions | Long steps = confusion or slow loading |
| Error rate | Assertions failed / total | High error = broken or confusing UI |
| Retry count | Same action repeated | User didn't understand the result |
| Back navigation | User went backwards | Flow didn't guide them forward |
| Help/tooltip usage | Clicked help icons | UI isn't self-explanatory |

### Friction Categories

Tag each friction point:

| Category | Example | Typical Fix |
|----------|---------|-------------|
| **Cognitive** | User doesn't know what to do next | Add guidance, progress bar |
| **Motor** | Too many clicks to reach goal | Reduce steps, add shortcuts |
| **Visual** | Can't find the button, low contrast | Improve visual hierarchy |
| **Technical** | Page loads slowly, form resets | Fix performance, add state persistence |
| **Trust** | User hesitates (payment, permissions) | Add reassurance (security badges, previews) |

---

## Pattern Detection

After 5+ sessions, analyze accumulated data for patterns:

### Long Flow Detection
```
Scan flows/ directory → sort by step count → flag flows with 7+ steps
→ For each: can steps be combined? Can defaults replace questions?
```

### Error Hotspots
```
Scan sessions/ directory → aggregate error locations
→ Which step in which flow fails most often?
→ Is it the same root cause (e.g., always OTP, always date picker)?
```

### Drop-off Risk Assessment
```
For each flow with 5+ steps:
→ Which step has the longest average time? (likely confusion point)
→ Which step has the most errors? (likely usability issue)
→ These are your top drop-off risks
```

### Quick Win Identification
```
Pain points where:
  Impact = High AND Effort = Low → Quick wins
  Impact = High AND Effort = Medium → Next sprint
  Impact = Low AND Effort = High → Deprioritize
```

---

## Metrics Dashboard (`metrics.md`)

Track trends across sessions:

```markdown
# QA Metrics — Trend Dashboard

## Flow Health Scores
| Flow | Current Steps | Target | Trend | Health |
|------|--------------|--------|-------|--------|
| Registration | 5 | 4 | Improving | Good |
| Checkout | 4 | 3 | Stable | Review |
| Settings update | 2 | 2 | Stable | Good |
| Report generation | 8 | 5 | Not started | Needs work |

## Session Frequency
| Month | Sessions Run | Flows Covered | Issues Found | Issues Fixed |
|-------|-------------|---------------|--------------|--------------|
| Mar 2026 | 12 | 6 | 18 | 11 |
| Apr 2026 | ... | ... | ... | ... |

## Top Unresolved Pain Points
1. Report generation flow — 8 steps, 3+ min, no progress indicator
2. Settings page — changes not saved notification missing
3. Mobile checkout — payment form resets on browser back
```

---

## README Template for Knowledge Base

When creating `e2e/knowledge/README.md` in a new project:

```markdown
# QA Knowledge Base

This directory contains structured data from QA sessions. It tracks user flows,
friction points, and improvement opportunities.

## How to Use

1. **Before building a feature** — check flows/ to understand the current state
2. **After QA testing** — add a session report and update relevant flow maps
3. **During sprint planning** — review improvements.md for data-driven priorities
4. **Monthly** — review metrics.md for trends and health scores

## Structure

- `flows/` — One file per user flow. Documents steps, time, variants, snapshots.
- `sessions/` — One file per QA session. Raw observations and metrics.
- `pain-points.md` — All friction points by severity. Append after each session.
- `improvements.md` — Prioritized improvement backlog from QA findings.
- `metrics.md` — Cross-session trends and flow health scores.

## Adding a New Flow Map

1. Run the QA session
2. Copy the flow map template from the ui-qa-testing skill
3. Fill in steps, time, errors, variants
4. Save as `flows/{flow-name}.flow.md`

## Analyzing Patterns

After 5+ sessions, look for:
- Flows with 7+ steps → simplification candidates
- Steps with highest error rate → usability issues
- Steps with longest time → confusion or performance problems
```
