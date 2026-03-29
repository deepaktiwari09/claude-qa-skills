# Bug Report Patterns

Templates for common bug categories with testable reproduction steps.

## Pattern 1: UI Rendering Bug

```markdown
## Bug Summary

Patient table shows overlapping text when patient name exceeds 30 characters on mobile viewport.

## User Impact

**Who:** All users on mobile/tablet (estimated 35% of traffic)
**Severity:** Degraded — data readable but requires horizontal scroll
**Frequency:** Always, for patients with long names
**Workaround:** Use desktop view

## Steps to Reproduce

### Prerequisites
- Logged in as any role
- Patient "Abdulrahman Mohammed Al-Qahtani" exists (name > 30 chars)

### Steps
1. Open /patients on mobile viewport (375px width)
2. Scroll to the patient with the long name
3. Observe the name column

### Expected Result
Name truncates with ellipsis (...) and shows full name on hover/tap

### Actual Result
Name overflows into the next column (MRN), making both unreadable

### Playwright CLI Reproduction
```bash
playwright-cli -s=bug open http://localhost:5173/patients --headed
playwright-cli -s=bug resize 375 812
playwright-cli -s=bug snapshot
playwright-cli -s=bug screenshot --filename=.playwright-cli/evidence/mobile-overflow-bug.png
```

## Environment
- Browser: Chrome 120 (mobile emulation), Safari iOS 17
- Viewport: 375x812 (iPhone SE)
- App: develop branch, commit abc1234
```

---

## Pattern 2: Data Integrity Bug

```markdown
## Bug Summary

Saving patient with Arabic name "أحمد" stores correctly but displays as "??????" after page reload.

## User Impact

**Who:** All users entering Arabic patient names (~60% of patient records)
**Severity:** Blocking — patient records appear corrupted
**Frequency:** Always with Arabic characters
**Workaround:** Enter names in English transliteration

## Steps to Reproduce

### Prerequisites
- Logged in as receptionist or doctor
- On patient creation form

### Steps
1. Navigate to /patients/new
2. Enter first name: أحمد
3. Enter last name: الراشد
4. Fill remaining required fields
5. Click Save
6. Observe success — name shows correctly
7. Navigate to /patients
8. Find the newly created patient
9. Observe the name column

### Expected Result
Name displays as "أحمد الراشد" everywhere

### Actual Result
Name shows as "?????? ??????" in the patient list table
Name shows correctly on patient detail page (inconsistent)

### Playwright CLI Reproduction
```bash
playwright-cli -s=bug open http://localhost:5173/patients/new --headed
playwright-cli -s=bug snapshot
playwright-cli -s=bug fill <first-name-ref> "أحمد"
playwright-cli -s=bug fill <last-name-ref> "الراشد"
# fill other required fields...
playwright-cli -s=bug click <save-ref>
playwright-cli -s=bug snapshot
playwright-cli -s=bug screenshot --filename=.playwright-cli/evidence/arabic-name-created.png
playwright-cli -s=bug goto http://localhost:5173/patients
playwright-cli -s=bug snapshot
playwright-cli -s=bug screenshot --filename=.playwright-cli/evidence/arabic-name-corrupted.png
```

## Possible Root Cause
- Table component may not be using UTF-8 rendering
- API response encoding differs between list and detail endpoints
- Check Content-Type header for charset=utf-8
```

---

## Pattern 3: Permission / Security Bug

```markdown
## Bug Summary

Nurse role can access /admin/users page by typing URL directly, bypassing sidebar restriction.

## User Impact

**Who:** Security concern — nurse role can see admin user list
**Severity:** Blocking (security)
**Frequency:** Always — deterministic
**Workaround:** None

## Steps to Reproduce

### Prerequisites
- Logged in as nurse role (NOT admin)

### Steps
1. Login as nurse (nurse@hospital.com)
2. Verify sidebar does NOT show "User Management" link — correct
3. Type http://localhost:5173/admin/users directly in browser
4. Observe the page loads with full user list

### Expected Result
Redirect to /dashboard with "Access Denied" toast notification

### Actual Result
Full /admin/users page loads, nurse can see all user accounts including emails and roles

### Playwright CLI Reproduction
```bash
# Login as nurse
playwright-cli -s=nurse open http://localhost:5173/login --headed
playwright-cli -s=nurse snapshot
playwright-cli -s=nurse fill <email-ref> "nurse@hospital.com"
playwright-cli -s=nurse fill <password-ref> "password123"
playwright-cli -s=nurse click <login-ref>
playwright-cli -s=nurse snapshot

# Try direct URL access
playwright-cli -s=nurse goto http://localhost:5173/admin/users
playwright-cli -s=nurse snapshot
playwright-cli -s=nurse screenshot --filename=.playwright-cli/evidence/nurse-sees-admin-page.png
# Check if page loaded (BUG) or redirected (FIXED)
```

## Possible Root Cause
- Route guard only checks sidebar visibility, not page-level access
- Missing middleware/guard on /admin/* routes
- Client-side routing allows access, need server-side check too
```

---

## Pattern 4: Performance Bug

```markdown
## Bug Summary

Dashboard takes 12 seconds to load when user has 500+ patients, renders progressively with layout shift.

## User Impact

**Who:** Doctors with large patient panels
**Severity:** Degraded — usable but frustrating
**Frequency:** Always for users with 500+ patients
**Workaround:** None

## Steps to Reproduce

### Prerequisites
- Logged in as doctor with 500+ assigned patients
- Fresh page load (clear cache)

### Steps
1. Login as doctor (doctor-highload@hospital.com)
2. Wait for dashboard to load
3. Observe loading time and layout behavior

### Expected Result
- Dashboard loads within 3 seconds
- Skeleton loaders prevent layout shift
- Stats load progressively but don't rearrange

### Actual Result
- Full page blank for 8 seconds
- Content appears all at once at 12 seconds
- Layout shifts when charts render after stats

### Playwright CLI Reproduction
```bash
playwright-cli -s=perf open http://localhost:5173/login --headed
playwright-cli -s=perf fill <email-ref> "doctor-highload@hospital.com"
playwright-cli -s=perf fill <password-ref> "password123"
playwright-cli -s=perf click <login-ref>
# Start tracing to capture timing
playwright-cli -s=perf tracing-start
playwright-cli -s=perf goto http://localhost:5173/dashboard
playwright-cli -s=perf tracing-stop
# Check network waterfall
playwright-cli -s=perf network
playwright-cli -s=perf screenshot --filename=.playwright-cli/evidence/slow-dashboard.png
```

## Possible Root Cause
- All API calls are sequential (waterfall), not parallel
- No pagination on patient stats aggregation endpoint
- Missing loading skeletons cause CLS
```

---

## Pattern 5: State Management Bug

```markdown
## Bug Summary

After editing a patient and navigating back to patient list, the list still shows stale (pre-edit) data until manual page refresh.

## User Impact

**Who:** Any user editing patient records
**Severity:** Degraded — data appears stale, causes confusion
**Frequency:** Always after edit
**Workaround:** Manual browser refresh (F5)

## Steps to Reproduce

### Steps
1. Go to /patients — note patient Ahmed's phone: +971-50-1111111
2. Click Ahmed's row to open detail
3. Click Edit, change phone to +971-50-2222222
4. Save — success toast appears, detail page shows new phone
5. Click browser Back or breadcrumb "Patients"
6. Find Ahmed's row in the table

### Expected Result
Phone column shows +971-50-2222222 (updated value)

### Actual Result
Phone column shows +971-50-1111111 (stale value)
Refreshing the page (F5) shows the correct value

### Playwright CLI Reproduction
```bash
playwright-cli -s=stale open http://localhost:5173/patients --headed
playwright-cli -s=stale snapshot
playwright-cli -s=stale screenshot --filename=.playwright-cli/evidence/before-edit.png
# Navigate to patient detail
playwright-cli -s=stale click <ahmed-row-ref>
playwright-cli -s=stale click <edit-button-ref>
playwright-cli -s=stale fill <phone-ref> "+971-50-2222222"
playwright-cli -s=stale click <save-ref>
playwright-cli -s=stale snapshot
# Go back to list
playwright-cli -s=stale go-back
playwright-cli -s=stale snapshot
playwright-cli -s=stale screenshot --filename=.playwright-cli/evidence/after-edit-stale.png
# Compare the two screenshots
```

## Possible Root Cause
- React Query cache not invalidated after mutation
- Missing `queryClient.invalidateQueries({ queryKey: ['patients'] })` in edit mutation's onSuccess
```
