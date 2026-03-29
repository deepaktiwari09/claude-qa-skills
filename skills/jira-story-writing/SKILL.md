---
name: jira-story-writing
description: Write QA-friendly Jira stories, bugs, and subtasks that express real user problems and are directly testable by both playwright-cli automation and human QA. Use when asked to "create a ticket", "write a story", "file a bug", "create subtasks", "make a Jira issue", or when creating any Jira ticket through MCP.
compatibility: Works with any Jira MCP server integration
metadata:
  author: deepaktiwari09
  version: "1.0.0"
---

# QA-Friendly Jira Story Writing

You are a product owner and QA lead combined. Every ticket you write must pass two tests:
1. **A developer** reads it and knows exactly what to build
2. **A QA tester** (human or playwright-cli) reads it and knows exactly how to verify it

## Before Writing Any Ticket

### Step 1: Ask the User

Clarify these before writing:
- **Which Jira MCP profile?** (personal, websenor, trudoc)
- **Project key?** (e.g., EMR)
- **Who is the real user?** Not "the system" — a real person with a real problem
- **What pain does this solve?** Not "add feature X" — what problem exists today?

### Step 2: Think Like the End User

Before writing, articulate the user's pain in plain language:

> "Dr. Fatima is in a consultation with a patient. She needs to quickly find the patient's
> previous lab results but currently has to scroll through 200+ records. She loses 2-3 minutes
> per consultation. She needs a search that finds results in under 2 seconds."

This becomes the story's soul. Every AC traces back to solving Dr. Fatima's problem.

## Story Format

### Title Convention
```
[Action verb] [what] [context/where]
```
Examples:
- "Search patients by name, MRN, or phone on patient list"
- "Display patient vitals history as timeline chart"
- "Block appointment booking for inactive patients"

Bad titles (too vague):
- ~~"Patient search"~~
- ~~"Fix dashboard"~~
- ~~"Update UI"~~

### Description Template

```markdown
## User Problem

[WHO] currently faces [PAIN POINT] when trying to [GOAL].
This causes [IMPACT — time lost, errors, frustration, revenue loss].

**Real-world scenario:** [Concrete example with a named persona]

## User Story

AS A [specific role — doctor/nurse/receptionist/admin/patient]
I WANT TO [specific action — not "manage" or "handle"]
SO THAT [specific benefit — tied to the pain point above]

## Acceptance Criteria

### AC1: [Short descriptive name]
**Given** [precondition — what state the app is in]
**When** [action — what the user does]
**Then** [outcome — what should happen]

**Test data:** [specific values QA can use]
**Verify:** [what to check in the UI]

### AC2: [Short descriptive name]
**Given** [precondition]
**When** [action]
**Then** [outcome]

...

## Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| [edge case 1] | [what should happen] |
| [edge case 2] | [what should happen] |
| [edge case 3] | [what should happen] |

## QA Testing Guide

### Prerequisites
- [ ] Test account: [role + credentials or how to get them]
- [ ] Test data: [what data must exist]
- [ ] Environment: [URL]

### Happy Path Test
1. [Step 1 — navigate to page]
2. [Step 2 — perform action]
3. [Step 3 — verify result]
   - Expected: [specific UI state]

### Negative Tests
1. [What happens with invalid input]
2. [What happens with no data]
3. [What happens without permission]

### Playwright CLI Quick Test
```bash
playwright-cli -s=qa open [url] --headed
playwright-cli -s=qa snapshot
# [specific commands to test this story]
```

## UI/UX Notes

- [Layout expectations, component placement]
- [Responsive behavior if applicable]
- [Accessibility requirements]

## API Dependencies

| Endpoint | Method | Purpose |
|----------|--------|---------|
| [/api/v1/...] | GET/POST | [what it does] |

## Out of Scope

- [What this story does NOT include]
- [Related work that belongs in a separate ticket]
```

## Bug Report Format

```markdown
## Bug Summary

[ONE sentence — what's broken from the user's perspective]

## User Impact

**Who:** [which users are affected]
**Severity:** [blocking / degraded / cosmetic]
**Frequency:** [always / intermittent / rare]
**Workaround:** [exists / none]

## Steps to Reproduce

### Prerequisites
- Logged in as: [role]
- Page: [URL]
- Test data: [what must exist]

### Steps
1. [Navigate to...]
2. [Click/type/do...]
3. [Observe...]

### Expected Result
[What SHOULD happen]

### Actual Result
[What ACTUALLY happens]

### Evidence
[Screenshot path, console error, network response]

### Playwright CLI Reproduction
```bash
playwright-cli -s=bug open [url] --headed
# exact commands to reproduce
playwright-cli -s=bug snapshot
playwright-cli -s=bug screenshot --filename=bug-evidence.png
```

## Environment

- Browser: [Chrome/Firefox/Safari + version]
- OS: [macOS/Windows/iOS/Android]
- App version: [branch or deployed version]
- API response: [if relevant, paste the response]

## Possible Root Cause

[Developer hypothesis if known]
```

## Subtask Format

Subtasks are implementation units. Keep them focused:

```markdown
## What to Implement

[2-3 sentences — exactly what code change is needed]

## Files to Touch

- `src/features/[module]/...` — [what to add/change]
- `src/types/...` — [type changes if any]

## Acceptance Criteria

### AC1: [name]
**Given** [state]
**When** [action]
**Then** [result]

## QA Verification

After implementation, verify:
- [ ] [checkpoint 1]
- [ ] [checkpoint 2]
- [ ] pnpm build — zero errors
- [ ] pnpm lint — zero errors
```

## Writing Principles

### 1. Real Users, Real Pain

**BAD:**
> "As a user, I want to search patients so that I can find them."

**GOOD:**
> "Dr. Fatima sees 40+ patients daily. Finding a returning patient's record takes 2-3 minutes
> of scrolling. She needs instant search by name or MRN to stay on schedule."
>
> AS A doctor
> I WANT TO search patients by name, MRN, or phone number
> SO THAT I can pull up any patient's record within 5 seconds during a consultation

### 2. Testable Acceptance Criteria

**BAD:**
> "Search should work properly"

**GOOD:**
> **AC1: Search by partial name**
> **Given** I am on the patient list page with 500+ patients
> **When** I type "Ahm" in the search field
> **Then** the table filters within 500ms to show only patients whose name contains "Ahm"
>
> **Test data:** Patient "Ahmed Al-Rashid" (MRN: PAT-001) must exist
> **Verify:** Table row count decreases, "Ahmed" visible in results

### 3. Edge Cases as First-Class Citizens

Don't leave QA guessing. Enumerate edge cases:

| Scenario | Expected |
|----------|----------|
| Search with no matching results | Show "No patients found" with clear search button |
| Search with special characters (!@#) | Sanitize input, no errors |
| Search field empty after clearing | Show full patient list |
| Network error during search | Show error toast, keep previous results |
| Search with exactly 1 character | No search triggered (min 2 chars) |

### 4. Include the "Why Not" Section

Explicitly state what's out of scope to prevent scope creep:
```
## Out of Scope
- Advanced filters (date range, status) — separate story EMR-XXXX
- Export search results to CSV — backlog
- Search by diagnosis/condition — requires backend work first
```

### 5. API Contract in the Ticket

If the story involves API calls, specify them:
```
## API Dependencies
| Endpoint | Method | Request | Response |
|----------|--------|---------|----------|
| /api/v1/patients | GET | ?search=Ahm&page=1&limit=20 | { data: Patient[], total: number } |
```

This lets QA verify both UI AND API behavior.

## Creating Tickets via Jira MCP

When creating the ticket through MCP:

1. **Ask which profile** — `mcp-jira-personal`, `mcp-jira-websenor`, or `mcp-jira-trudoc`
2. **Use the correct tool** — `mcp__mcp-jira-{profile}__jira_create_issue`
3. **Format description as Markdown** — Jira renders it correctly
4. **Set appropriate fields:**
   - `issue_type`: "Story", "Bug", "Subtask", "Task"
   - `assignee`: ask user
   - `additional_fields`: labels, priority, sprint, epic link

### Story Creation Example

```
project_key: "EMR"
summary: "Search patients by name, MRN, or phone on patient list"
issue_type: "Story"
description: [full template above in markdown]
additional_fields: {"labels": ["frontend", "patient-module"], "priority": {"name": "High"}}
```

### Creating Subtasks Under a Story

After creating the parent story (e.g., EMR-11300):
```
project_key: "EMR"
summary: "Add search input with debounce to PatientTable"
issue_type: "Subtask"
description: [subtask template]
additional_fields: {"parent": "EMR-11300"}
```

## Quality Checklist

Before submitting any ticket, verify:

- [ ] **User pain is clear** — reader understands WHY this matters
- [ ] **Every AC has Given/When/Then** — no ambiguous criteria
- [ ] **Test data specified** — QA knows what data to use
- [ ] **Edge cases listed** — at least 3 edge cases per story
- [ ] **Out of scope defined** — prevents scope creep
- [ ] **Playwright CLI commands included** — automatable by AI
- [ ] **API endpoints listed** — if applicable
- [ ] **No jargon without context** — new team member could understand it

## Specific References

* **Story examples by domain** [references/story-examples.md](references/story-examples.md)
* **Bug report patterns** [references/bug-patterns.md](references/bug-patterns.md)
