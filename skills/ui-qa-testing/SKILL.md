---
name: ui-qa-testing
description: End-to-end UI QA testing using playwright-cli. Use when asked to test user flows, validate user stories, QA a feature, run acceptance tests, verify UI behavior, test as different user roles, or generate QA evidence reports. Triggers on "test this flow", "QA this feature", "verify the user story", "test as [role]", "run acceptance tests", "check if this works".
compatibility: Requires playwright-cli (npm install -g @playwright/cli)
allowed-tools: Bash(playwright-cli:*)
metadata:
  author: deepaktiwari09
  version: "1.0.0"
---

# UI QA Testing with playwright-cli

You are a senior QA engineer. Your job is to validate that the application works correctly from the end user's perspective by driving a real browser.

## Prerequisites

This skill requires `playwright-cli` installed globally:
```bash
playwright-cli --version
```

If the project has a local `.claude/skills/playwright-cli/SKILL.md`, read it for command reference. Otherwise use `playwright-cli --help`.

## Core Workflow

### Phase 1: Understand What to Test

Before touching the browser, gather context:

1. **Read the user story / acceptance criteria** — ask the user or check Jira
2. **Identify the app URL** — local dev server (usually `http://localhost:5173` or similar)
3. **Identify test roles** — which user types interact with this feature?
4. **List the happy path** — the primary flow that must work
5. **List edge cases** — empty states, validation errors, permission denied, slow network

Structure your test plan as a checklist:
```
## Test Plan: [Feature Name]
- [ ] Happy path: [describe primary flow]
- [ ] Validation: [form errors, required fields]
- [ ] Empty state: [no data scenario]
- [ ] Role-based: [what happens for unauthorized users]
- [ ] Responsive: [mobile/tablet if applicable]
```

### Phase 2: Set Up Browser Session

```bash
# Use named session matching the feature being tested
playwright-cli -s=qa open http://localhost:5173 --headed

# For auth-required apps, login first then save state
playwright-cli -s=qa snapshot
playwright-cli -s=qa fill <email-ref> "test@example.com"
playwright-cli -s=qa fill <password-ref> "password"
playwright-cli -s=qa click <login-button-ref>
playwright-cli -s=qa state-save .playwright-cli/auth-state.json
```

### Phase 3: Execute Tests Systematically

For each test case:

1. **Navigate** to the starting point
2. **Snapshot** to understand the page
3. **Act** — perform the user actions
4. **Snapshot again** to verify the result
5. **Screenshot** key states as evidence
6. **Record verdict** — PASS / FAIL with reason

```bash
# Example: Testing a search feature
playwright-cli -s=qa goto http://localhost:5173/patients
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/01-patient-list-initial.png

# Act: search for a patient
playwright-cli -s=qa fill <search-ref> "Ahmed"
playwright-cli -s=qa press Enter
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/02-search-results.png

# Verify: check the snapshot for expected results
# The snapshot YAML will show table rows — check if "Ahmed" appears
```

### Phase 4: Generate QA Report

After all tests, create a markdown report:

```markdown
# QA Report: [Feature Name]
**Date:** YYYY-MM-DD
**Tester:** Claude Code (automated)
**App URL:** http://localhost:5173
**Branch:** [current git branch]

## Summary
- Total tests: X
- Passed: Y
- Failed: Z

## Test Results

### 1. [Test Case Name] — PASS/FAIL
**Steps:**
1. Navigate to /patients
2. Enter "Ahmed" in search
3. Press Enter

**Expected:** Table filters to show patients matching "Ahmed"
**Actual:** [what happened]
**Evidence:** [screenshot path]

### 2. [Next test case...]
```

Save the report to `.playwright-cli/qa-reports/[feature]-[date].md`.

## Testing Patterns

### Pattern: User Story Acceptance Testing

When given a Jira user story with acceptance criteria:

```
AS A doctor
I WANT TO search patients by name
SO THAT I can quickly find patient records

Acceptance Criteria:
- [ ] Search field visible on patient list page
- [ ] Typing filters results in real-time
- [ ] No results shows empty state message
- [ ] Clearing search shows all patients
```

Map each AC to a test case. Execute each one. Report with evidence.

### Pattern: Multi-Role Testing

Test the same feature as different user types using sessions:

```bash
# Session 1: Test as admin
playwright-cli -s=admin open http://localhost:5173
# ... login as admin, test feature ...
playwright-cli -s=admin state-save .playwright-cli/admin-auth.json

# Session 2: Test as regular user
playwright-cli -s=user open http://localhost:5173
# ... login as regular user, test same feature ...

# Session 3: Test as unauthenticated
playwright-cli -s=guest open http://localhost:5173
# ... verify access is denied ...
```

### Pattern: Form Validation Testing

```bash
# Test empty submission
playwright-cli -s=qa click <submit-ref>
playwright-cli -s=qa snapshot
# Check snapshot for error messages

# Test invalid input
playwright-cli -s=qa fill <email-ref> "not-an-email"
playwright-cli -s=qa click <submit-ref>
playwright-cli -s=qa snapshot

# Test valid input
playwright-cli -s=qa fill <email-ref> "valid@email.com"
playwright-cli -s=qa click <submit-ref>
playwright-cli -s=qa snapshot
```

### Pattern: CRUD Flow Testing

Test full Create → Read → Update → Delete cycle:

```bash
# CREATE
playwright-cli -s=qa goto http://localhost:5173/items/new
playwright-cli -s=qa snapshot
playwright-cli -s=qa fill <name-ref> "Test Item"
playwright-cli -s=qa click <save-ref>
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/crud-01-created.png

# READ — verify item appears in list
playwright-cli -s=qa goto http://localhost:5173/items
playwright-cli -s=qa snapshot
# Check snapshot for "Test Item"

# UPDATE
playwright-cli -s=qa click <edit-ref>
playwright-cli -s=qa fill <name-ref> "Updated Item"
playwright-cli -s=qa click <save-ref>
playwright-cli -s=qa snapshot

# DELETE
playwright-cli -s=qa click <delete-ref>
playwright-cli -s=qa snapshot
# Check confirmation dialog
playwright-cli -s=qa click <confirm-ref>
playwright-cli -s=qa snapshot
```

### Pattern: Navigation & Routing

```bash
# Test breadcrumb navigation
playwright-cli -s=qa goto http://localhost:5173/patients/123
playwright-cli -s=qa snapshot
# Click breadcrumb "Patients"
playwright-cli -s=qa click <breadcrumb-ref>
playwright-cli -s=qa snapshot
# Verify URL changed to /patients

# Test browser back/forward
playwright-cli -s=qa go-back
playwright-cli -s=qa snapshot

# Test direct URL access
playwright-cli -s=qa goto http://localhost:5173/some/deep/route
playwright-cli -s=qa snapshot
```

### Pattern: Network & API Verification

```bash
# Clear network log, perform action, check API calls
playwright-cli -s=qa goto http://localhost:5173/dashboard
playwright-cli -s=qa network
# Review network log for expected API calls, status codes, timing
```

### Pattern: Error State Testing

```bash
# Mock API failure
playwright-cli -s=qa route "**/api/v1/patients" --status=500 --body='{"error":"Internal Server Error"}'
playwright-cli -s=qa goto http://localhost:5173/patients
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/error-state-500.png
# Verify error UI shows correctly

# Remove mock
playwright-cli -s=qa unroute "**/api/v1/patients"

# Mock empty response
playwright-cli -s=qa route "**/api/v1/patients" --body='{"data":[],"total":0}' --content-type=application/json
playwright-cli -s=qa reload
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/empty-state.png
playwright-cli -s=qa unroute
```

### Pattern: Video Evidence for Complex Flows

```bash
playwright-cli -s=qa video-start
# ... perform entire user flow ...
playwright-cli -s=qa video-stop .playwright-cli/evidence/login-to-appointment-flow.webm
```

## Evidence Organization

Always organize evidence in `.playwright-cli/evidence/`:
```
.playwright-cli/
  evidence/
    [feature-name]/
      01-initial-state.png
      02-after-action.png
      03-result.png
      flow-recording.webm
  qa-reports/
    [feature]-[date].md
  auth-state.json
```

Add to `.gitignore`:
```
.playwright-cli/
```

## Cleanup

Always close browser sessions when done:
```bash
playwright-cli -s=qa close
# or close all
playwright-cli close-all
```

## Quick Reference

| Task | Command |
|------|---------|
| Start QA session | `playwright-cli -s=qa open <url> --headed` |
| See page structure | `playwright-cli -s=qa snapshot` |
| Click element | `playwright-cli -s=qa click <ref>` |
| Fill input | `playwright-cli -s=qa fill <ref> "text"` |
| Take evidence | `playwright-cli -s=qa screenshot --filename=path.png` |
| Check API calls | `playwright-cli -s=qa network` |
| Mock API error | `playwright-cli -s=qa route "<pattern>" --status=500` |
| Save login state | `playwright-cli -s=qa state-save auth.json` |
| Record video | `playwright-cli -s=qa video-start` / `video-stop` |
| End session | `playwright-cli -s=qa close` |

## Specific Tasks

* **User story acceptance testing** [references/user-story-testing.md](references/user-story-testing.md)
* **Generating E2E test specs** [references/e2e-spec-generation.md](references/e2e-spec-generation.md)
* **Self-QA after coding** [references/self-qa-workflow.md](references/self-qa-workflow.md)
* **QA knowledge base** [references/knowledge-base.md](references/knowledge-base.md)
