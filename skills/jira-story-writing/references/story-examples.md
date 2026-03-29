# Story Examples by Domain

Real-world story examples showing how to express user pain and write testable criteria.

## Healthcare / EMR Domain

### Example 1: Patient Search

```markdown
## User Problem

Dr. Fatima sees 40+ patients daily at TruDoc clinic. When a returning patient walks in,
she has to scroll through 500+ patient records to find them. This takes 2-3 minutes per
consultation, adding up to ~90 minutes of wasted time daily across her patient load.
Nurses at the front desk face the same issue during check-in.

**Real-world scenario:** Patient Ahmed Al-Rashid (MRN: PAT-00142) arrives for a follow-up.
Dr. Fatima remembers his name but not his MRN. She needs to find his record before the
consultation timer starts.

## User Story

AS A doctor
I WANT TO search patients by name, MRN, or phone number with instant results
SO THAT I can pull up any patient's record within 5 seconds during a consultation

## Acceptance Criteria

### AC1: Search by partial name
**Given** I am on the patient list page (/patients)
**When** I type "Ahm" in the search field
**Then** the table filters within 500ms to show only patients whose first or last name contains "Ahm"

**Test data:** Patient "Ahmed Al-Rashid" (MRN: PAT-00142, phone: +971-50-1234567)
**Verify:** Table shows Ahmed's row, total count updates

### AC2: Search by MRN
**Given** I am on the patient list page
**When** I type "PAT-001" in the search field
**Then** patients whose MRN starts with "PAT-001" appear

**Test data:** MRN "PAT-00142"
**Verify:** Ahmed's row appears

### AC3: Search by phone number
**Given** I am on the patient list page
**When** I type "050123" in the search field
**Then** patients whose phone contains "050123" appear

### AC4: Clear search restores full list
**Given** I have searched for "Ahmed" and see filtered results
**When** I click the clear (X) button in the search field
**Then** the full patient list is restored with original pagination

### AC5: No results state
**Given** I am on the patient list page
**When** I search for "xyznonexistent"
**Then** I see an empty state with message "No patients found" and a "Clear search" link

## Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| Type only 1 character | No search triggered — minimum 2 chars |
| Special characters (!@#$%) | Input sanitized, no error |
| Arabic name search (أحمد) | Works with Arabic characters |
| Very long search string (100+ chars) | Truncated to 50 chars |
| Network timeout during search | Show error toast, keep previous results |
| Search while page is loading | Queue search, execute after load |

## QA Testing Guide

### Prerequisites
- [ ] Logged in as doctor role
- [ ] At least 20 patients exist in the system
- [ ] Patient "Ahmed Al-Rashid" exists with MRN PAT-00142

### Happy Path
1. Go to /patients
2. Type "Ahmed" in search — verify table filters
3. Clear search — verify full list returns
4. Type "PAT-001" — verify MRN search works
5. Type "050123" — verify phone search works

### Playwright CLI Quick Test
```bash
playwright-cli -s=qa open http://localhost:5173/patients --headed
playwright-cli -s=qa snapshot
# Find search input ref from snapshot
playwright-cli -s=qa fill <search-ref> "Ahmed"
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/search-ahmed.png
# Clear and test no results
playwright-cli -s=qa fill <search-ref> "xyznonexistent"
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/search-no-results.png
```

## API Dependencies

| Endpoint | Method | Params | Response |
|----------|--------|--------|----------|
| /api/v1/patients | GET | ?search=Ahmed&page=1&limit=20 | { data: Patient[], total: number, page: number } |

## Out of Scope
- Filter by date of birth, gender, status — separate story
- Export search results — backlog
- Recent searches history — future enhancement
```

---

## Example 2: Appointment Booking Conflict

```markdown
## User Problem

Receptionist Mariam books 60+ appointments daily. Last week, she accidentally double-booked
Dr. Hassan at 10:00 AM — two patients showed up for the same slot, one had to wait 45 minutes.
This happened because the booking form doesn't warn about conflicts in real-time.

**Real-world scenario:** Patient Khalid calls to book with Dr. Hassan on Tuesday at 10:00 AM.
Mariam doesn't know that Sarah already has that slot. She books it, both patients get confirmation
SMS. On Tuesday, chaos.

## User Story

AS A receptionist
I WANT TO see real-time availability when booking an appointment
SO THAT I never double-book a doctor's time slot

## Acceptance Criteria

### AC1: Show available slots only
**Given** I am on the appointment booking form
**When** I select Dr. Hassan and date Tuesday March 25
**Then** I see only available time slots (occupied slots are greyed out with patient initials)

**Test data:** Dr. Hassan has appointments at 9:00, 10:00, 11:30 on March 25
**Verify:** Those 3 slots show as unavailable, remaining slots are clickable

### AC2: Conflict warning on overlapping booking
**Given** Dr. Hassan already has an appointment at 10:00 AM
**When** I try to submit a new booking for Dr. Hassan at 10:00 AM
**Then** I see a blocking warning: "Dr. Hassan already has an appointment with [Patient Name] at this time"
**And** the submit button is disabled

### AC3: Allow booking adjacent slots
**Given** Dr. Hassan has appointment at 10:00-10:30 AM
**When** I try to book 10:30 AM
**Then** booking succeeds (no overlap)

## Edge Cases

| Scenario | Expected |
|----------|----------|
| Two receptionists booking same slot simultaneously | First submission wins, second gets conflict error |
| Booking for cancelled appointment's slot | Slot is available again |
| Doctor's day off | All slots greyed with "Day Off" label |
| Past date selected | Date picker blocks past dates |

## QA Testing Guide

### Prerequisites
- [ ] Logged in as receptionist
- [ ] Dr. Hassan exists with working hours 9:00-17:00
- [ ] At least 3 existing appointments for test date

### Playwright CLI Quick Test
```bash
playwright-cli -s=qa open http://localhost:5173/appointments/new --headed
playwright-cli -s=qa snapshot
# Select doctor
playwright-cli -s=qa select <doctor-ref> "Dr. Hassan"
# Select date
playwright-cli -s=qa click <date-ref>
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/available-slots.png
# Try to book occupied slot
playwright-cli -s=qa click <occupied-slot-ref>
playwright-cli -s=qa snapshot
playwright-cli -s=qa screenshot --filename=.playwright-cli/evidence/conflict-warning.png
```
```

---

## E-Commerce Domain

### Example 3: Cart Abandonment Recovery

```markdown
## User Problem

Analytics show 68% cart abandonment rate. Users add items but leave at checkout.
Exit surveys reveal: "I wasn't sure about shipping cost" (42%), "Process too long" (31%),
"Had to create account" (27%). We're losing ~$12K/month in recoverable revenue.

**Real-world scenario:** Sarah adds a $89 dress to cart. She proceeds to checkout but sees
shipping is calculated only AFTER entering her address. She abandons. 2 hours later, she
buys the same dress from a competitor that showed "Free shipping over $75" upfront.

## User Story

AS A shopper
I WANT TO see the total cost including shipping before starting checkout
SO THAT I can make a purchase decision without surprises

## Acceptance Criteria

### AC1: Show estimated shipping on cart page
**Given** I have items in my cart totaling $89
**When** I view my cart
**Then** I see "Estimated shipping: Free (orders over $75)" or "Estimated shipping: $5.99"

### AC2: Free shipping threshold indicator
**Given** my cart total is $62 (below $75 threshold)
**When** I view my cart
**Then** I see "Add $13 more for free shipping" with a progress bar

### AC3: Shipping updates with cart changes
**Given** I am on the cart page showing "Free shipping"
**When** I remove an item bringing total below $75
**Then** shipping estimate updates to "$5.99" and progress bar appears

## Edge Cases

| Scenario | Expected |
|----------|----------|
| Empty cart | No shipping info shown |
| International address detected | Show "International shipping calculated at checkout" |
| All items are digital | Show "No shipping needed — digital delivery" |
| Cart total exactly $75.00 | Free shipping qualifies |
```

---

## SaaS / Internal Tool Domain

### Example 4: Role-Based Dashboard

```markdown
## User Problem

All 45 employees see the same dashboard regardless of role. The CEO sees ticket counts she
doesn't need. Support agents see revenue charts they can't act on. Everyone ignores the
dashboard because 80% of it isn't relevant to them.

**Real-world scenario:** Support lead Omar opens the dashboard every morning to check
open ticket count and average response time. These two numbers are buried below revenue
charts and marketing metrics. He spends 30 seconds scrolling past irrelevant data every time.

## User Story

AS A support team lead
I WANT TO see only support-relevant metrics on my dashboard
SO THAT I can assess team performance in under 5 seconds upon login

## Acceptance Criteria

### AC1: Dashboard shows role-appropriate widgets
**Given** I am logged in as a user with "support_lead" role
**When** the dashboard loads
**Then** I see: Open Tickets, Avg Response Time, SLA Compliance, Ticket Trend chart
**And** I do NOT see: Revenue, Marketing, HR widgets

**Test data:** User with support_lead role, 25 open tickets, 4.2hr avg response
**Verify:** Exactly 4 widgets visible, values match test data

### AC2: Admin sees all widgets
**Given** I am logged in as "admin" role
**When** the dashboard loads
**Then** I see all widget categories with a role filter dropdown

### AC3: Unknown role gets default view
**Given** I am logged in with a role that has no dashboard config
**When** the dashboard loads
**Then** I see a generic dashboard with company-wide metrics
```
