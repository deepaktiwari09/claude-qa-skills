# Self-QA After Coding

Workflow for Claude Code to self-test features immediately after implementing them.

## When to Use

After implementing a feature or fixing a bug, before committing:

1. Build passes (`pnpm build`)
2. Dev server is running (`pnpm dev`)
3. Use this workflow to verify the UI works as expected

## Quick Self-QA Flow

```bash
# 1. Open the affected page
playwright-cli -s=qa open http://localhost:5173/<page> --headed

# 2. Snapshot to verify page loaded
playwright-cli -s=qa snapshot

# 3. Test the specific change
# ... interact with the feature you just built ...

# 4. Screenshot as evidence
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/self-qa-<feature>.png

# 5. Check console for errors
playwright-cli -s=qa console error

# 6. Check network for failed requests
playwright-cli -s=qa network

# 7. Clean up
playwright-cli -s=qa close
```

## What to Check

### After Adding a New Component
- [ ] Component renders without console errors
- [ ] Component is interactive (buttons click, inputs accept text)
- [ ] Component shows correct data
- [ ] Component handles empty/loading state

### After Fixing a Bug
- [ ] The bug is actually fixed (reproduce original steps)
- [ ] No regression in surrounding functionality
- [ ] No new console errors

### After Adding a New Page/Route
- [ ] Page loads at expected URL
- [ ] Page shows in navigation/sidebar
- [ ] Breadcrumbs work
- [ ] Browser back/forward works
- [ ] Direct URL access works

### After Modifying API Integration
- [ ] API call fires correctly (check network)
- [ ] Loading state shows during fetch
- [ ] Data renders correctly
- [ ] Error state works (mock 500 with route)

## Reporting Self-QA Results

After self-QA, report to the user concisely:

```markdown
**Self-QA Results:**
- Page loads: OK
- Feature works: OK (searched "Ahmed", table filtered correctly)
- Console errors: None
- Network: All API calls 200 OK
- Evidence: .playwright-cli/evidence/self-qa-patient-search.png
```

## Combining with Build Verification

Full pre-commit verification:
```bash
# 1. Code quality
pnpm lint
pnpm build

# 2. Unit tests
pnpm test

# 3. Self-QA (if dev server is running)
playwright-cli -s=qa open http://localhost:5173/<affected-page> --headed
# ... test the change ...
playwright-cli -s=qa close
```
