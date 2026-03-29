# User Story Acceptance Testing

Systematic workflow for testing Jira user stories against their acceptance criteria.

## Workflow

### Step 1: Parse the User Story

Extract from the story:
- **Role**: Who is the user? (doctor, nurse, admin, patient)
- **Goal**: What do they want to do?
- **Benefit**: Why? What problem does this solve?
- **Acceptance Criteria**: Specific conditions that must be true

### Step 2: Create Test Matrix

For each acceptance criterion, create a test case:

```markdown
| AC# | Criterion | Test Steps | Expected Result | Status |
|-----|-----------|------------|-----------------|--------|
| AC1 | Search filters in real-time | Type "Ahm" in search | Table shows matching rows | |
| AC2 | Empty search shows all | Clear search field | All patients visible | |
| AC3 | No results shows message | Search "zzzzz" | "No patients found" shown | |
```

### Step 3: Execute Each Test Case

```bash
# Setup
playwright-cli -s=qa open http://localhost:5173 --headed

# For each AC:
# 1. Navigate to starting point
playwright-cli -s=qa goto <url>

# 2. Take "before" snapshot
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/AC1-before.png

# 3. Perform actions
playwright-cli -s=qa fill <ref> "search term"

# 4. Take "after" snapshot
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/AC1-after.png

# 5. Analyze snapshot to determine PASS/FAIL
# Read the snapshot YAML to verify expected elements exist
```

### Step 4: Report Results

Generate a structured report mapping each AC to its result:

```markdown
# Acceptance Test Report
**Story:** EMR-XXXXX — [Story Title]
**Date:** YYYY-MM-DD
**Environment:** http://localhost:5173 (branch: feature/xyz)

## Acceptance Criteria Results

### AC1: Search filters in real-time — PASS
**Steps:** Typed "Ahm" in search field on /patients
**Expected:** Table shows only matching patients
**Actual:** Table filtered to 3 results containing "Ahmed"
**Evidence:** AC1-before.png, AC1-after.png

### AC2: Empty search shows all — PASS
...

## Summary
- AC1: PASS
- AC2: PASS
- AC3: FAIL — Empty state message missing, shows blank table instead
```

## Testing Negative Scenarios

For each AC, also test the inverse:

| Positive Test | Negative Test |
|---------------|---------------|
| Valid form submits | Empty form shows errors |
| Authorized user sees page | Unauthorized gets redirected |
| Search finds results | Search with no match shows empty state |
| File upload succeeds | Invalid file type shows error |

## Multi-Role Testing for Stories

If the story specifies "AS A [role]", verify:

1. **Authorized role** can perform the action
2. **Other roles** cannot (or see appropriate restrictions)
3. **Unauthenticated users** are redirected to login

```bash
# Test as the intended role
playwright-cli -s=doctor open http://localhost:5173
# ... login as doctor, verify feature works ...

# Test as a different role
playwright-cli -s=nurse open http://localhost:5173
# ... login as nurse, verify appropriate access ...

# Test unauthenticated
playwright-cli -s=guest open http://localhost:5173/protected-page
# ... verify redirect to login ...
```

## Real-World Example: Patient Search Story

```
AS A doctor
I WANT TO search patients by name, MRN, or phone
SO THAT I can quickly find patient records during consultation

AC1: Search bar visible at top of patient list
AC2: Typing 3+ characters triggers search
AC3: Results update within 500ms
AC4: Matching text highlighted in results
AC5: Clear button resets to full list
AC6: No results shows helpful message
```

Test execution:
```bash
playwright-cli -s=qa open http://localhost:5173/patients --headed

# AC1: Search bar visible
playwright-cli -s=qa snapshot
# Verify snapshot contains search input element

# AC2: Typing triggers search
playwright-cli -s=qa fill <search-ref> "Ahm"
playwright-cli -s=qa snapshot
# Verify table changed

# AC3: Response time (check network)
playwright-cli -s=qa network
# Check API call timing

# AC4: Highlighting (inspect DOM)
playwright-cli -s=qa eval "document.querySelectorAll('mark, .highlight').length"

# AC5: Clear button
playwright-cli -s=qa click <clear-ref>
playwright-cli -s=qa snapshot
# Verify full list restored

# AC6: No results
playwright-cli -s=qa fill <search-ref> "xyznonexistent"
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/AC6-no-results.png
# Verify empty state message
```
