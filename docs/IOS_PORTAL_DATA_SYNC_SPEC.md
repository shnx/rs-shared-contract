# iOS Portal Data Sync Specification — Website Project

> **From:** Web Team
> **To:** iOS Team
> **Subject:** Complete portal structure, tab definitions, data sources, and fetching logic for the "website" project type — for building a native iOS mirror

---

## 1. Architecture Overview

### 1.1 Tech Stack

- **Frontend:** React + TypeScript + Vite
- **Backend:** Firebase (Firestore + Auth + Cloud Functions)
- **Database:** Firestore (collections listed below)
- **Auth:** Firebase Auth (email/password, Google, Apple)
- **Email:** Resend (via Cloud Functions) + `email_logs` collection
- **Payments:** PayPal (Cloud Functions), Tap, Stripe
- **Calendar:** Google Calendar (via Cloud Functions)

### 1.2 Firebase Configuration

All Firebase config is in `src/utils/firebase.ts`. The iOS app should use the **same Firebase project** with the same collections.

**Firebase project ID:** `robotics-website-5593f`

**Key exports:**
- `firebaseDB` — generic CRUD wrapper around Firestore (getAll, getById, create, update, delete, query)
- `firebaseAuth` — auth wrapper (signIn, signUp, createUserAsAdmin, sendPasswordReset, Google/Apple sign-in)
- `db` — raw Firestore instance

### 1.3 Data Access Pattern

Every page follows the same pattern:
1. `useEffect` on mount calls a `loadData()` async function
2. `loadData()` calls one or more `xxxDB.getAll()` (or `Promise.all([...])`)
3. Each `xxxDB` is a thin wrapper around `firebaseDB.getAll(STORAGE_KEYS.XXX)` that normalizes dates
4. Results stored in `useState`, then filtered/sorted with `useMemo`
5. Mutations call `xxxDB.update()` / `xxxDB.create()` / `xxxDB.delete()`, then `loadData()` again

**For iOS:** Replicate this by reading the same Firestore collections directly. The `xxxDB` wrappers just normalize Firestore timestamps (`{seconds, nanoseconds}`) into JS `Date` objects. Do the same in Swift.

### 1.4 Firestore Collections — Complete Reference

| Collection Name | Constant in Code | Used By |
|---|---|---|
| `systemUsers` | — | People, Auth, all pages |
| `job_submissions` | `JOB_SUBMISSIONS` | People, Inbox |
| `crm_contacts` | — | People |
| `student_crm` | `STUDENT_CRM` | People |
| `bookings` | `BOOKINGS` | Calendar, Bookings, Inbox, PayPal |
| `contact_submissions` | — | People, Website Manager |
| `call_invitations` | `CALL_INVITATIONS` | People, Invitations |
| `portal_messages` | — | People, Communication |
| `email_logs` | `EMAIL_LOGS` | People, Communication |
| `analytics_events` | `ANALYTICS_EVENTS` | People, Analytics |
| `analytics_sessions` | `ANALYTICS_SESSIONS` | People, Analytics |
| `service_offerings` | `SERVICE_OFFERINGS` | Calendar, Bookings, Service Offerings |
| `availability` | `AVAILABILITY` | Calendar |
| `slot_overrides` | `SLOT_OVERRIDES` | Calendar |
| `calendar_blocks` | `CALENDAR_BLOCKS` | Calendar |
| `slot_groups` | `SLOT_GROUPS` | Calendar |
| `session_outcomes` | `SESSION_OUTCOMES` | Session Outcomes |
| `coupons` | `COUPONS` | Invitations & Coupons, Bookings |
| `site_config` | — | Calendar config, Booking Preview |
| `projects` | `PROJECTS` | Projects |
| `documents` | `DOCUMENTS` | Documents |
| `user_messages` | `USER_MESSAGES` | User Messages, Inbox |
| `communication_status` | `COMMUNICATION_STATUS` | People (merge) |
| `email_responses` | `EMAIL_RESPONSES` | People |
| `candidate_posts` | — | People (merge) |
| `discover_quiz_results` | `DISCOVER_QUIZ_RESULTS` | Analytics |
| `paypal_config` | — | PayPal, Admin Settings |
| `tap_config` | — | Tap Config |
| `audit_logs` | `AUDIT_LOGS` | Admin Settings |
| `shared_links` | `SHARED_LINKS` | Link Manager |
| `career_opportunities` | — | Website Manager |
| `price_escalation_rules` | — | Price Escalation |
| `proposals` | — | Proposal Generator |
| `membership_plans` | — | (legacy, may still exist) |

---

## 2. Tab System

### 2.1 Tab Registry

Tabs are registered in `src/tabs/tabRegistry.ts` and individual files in `src/tabs/*.tsx`. Each tab calls `registerTab({...})` at module load time.

**TabDefinition fields:**
- `id` — unique identifier (used in URLs and navigation)
- `label` / `labelAr` — English and Arabic labels
- `icon` — Lucide icon component
- `component` — React component (lazy-loaded)
- `roles` — which user roles can see this tab
- `projectTypes` — which project types show this tab
- `category` — sidebar grouping (overview, people, finance, bookings, operations, website, admin)
- `layout` — `'widget-grid'` or `'custom'`

**Tab filtering:** `getTabsForProject('website')` returns all tabs where `projectTypes` includes `'website'`. Then the sidebar filters by the user's role.

### 2.2 Sidebar Categories (in order)

| Category | Label |
|---|---|
| `overview` | Dashboard |
| `people` | People |
| `finance` | Finance |
| `bookings` | Bookings & Services |
| `operations` | Operations |
| `website` | Website |
| `admin` | Settings |

### 2.3 Complete Active Tabs for Website Project

Here is every tab that appears for the `website` project type, organized by sidebar category:

#### Dashboard (overview)

| # | Tab ID | Label | Component | Roles |
|---|---|---|---|---|
| 1 | `inbox` | Inbox | `InboxDashboard` | admin, manager, advisor |
| 2 | `documents` | Documents | `Documents` | admin, manager, advisor, client |
| 3 | `analytics` | Analytics | `AdminAnalytics` | admin, manager |

#### People

| # | Tab ID | Label | Component | Roles |
|---|---|---|---|---|
| 4 | `people` | People | `PeopleHub` | admin, manager |
| 5 | `communication` | Communication | `CommunicationManagement` | admin, manager |
| 6 | `user-messages` | Messages to Team | `UserMessages` | student, candidate, client, advisor |

#### Finance

| # | Tab ID | Label | Component | Roles |
|---|---|---|---|---|
| 7 | `paypal-payments` | PayPal Payments | `PayPalPayments` | admin, manager, advisor |

#### Bookings & Services

| # | Tab ID | Label | Component | Roles |
|---|---|---|---|---|
| 8 | `calendar` | Calendar | `CalendarManagement` | admin, manager, advisor |
| 9 | `bookings` | Bookings | `BookingsManagement` | admin, manager, advisor |
| 10 | `invitations-coupons` | Invitations & Coupons | `InvitationsCoupons` | admin, manager, advisor |
| 11 | `service-offerings` | Service Offerings | `ServiceOfferingsManagement` | admin, manager |
| 12 | `session-outcomes` | Session Outcomes | `OutcomeManagement` | admin, manager |
| 13 | `price-escalation` | Price Escalation | `PriceEscalationManagement` | admin, manager |
| 14 | `proposal-generator` | Proposal Generator | `ProposalGenerator` | admin, manager, advisor |
| 15 | `booking-preview` | Booking Preview | `BookingPreview` | admin, manager |
| 16 | `website-manager` | Website Manager | `WebsiteManager` | admin, manager |

#### Settings (admin)

| # | Tab ID | Label | Component | Roles |
|---|---|---|---|---|
| 17 | `projects` | Projects | `ProjectManagement` | admin, manager |
| 18 | `tap-config` | Tap Config | `TapConfigManagement` | admin |
| 19 | `admin-settings` | Admin Settings | `AdminSettings` | admin |
| 20 | `link-manager` | Link Manager | `LinkManager` | admin, manager |
| 21 | `help-guide` | Help Guide | `HelpGuide` | admin, manager, advisor |

---

## 3. Tab-by-Tab Data Fetching Guide

### 3.1 Tab: `inbox` — InboxDashboard

**Purpose:** Unified admin inbox showing items needing attention: pending bookings, new CV submissions, unread user messages, and today's bookings.

**File:** `src/pages/InboxDashboard.tsx`

**Data fetched on mount (`loadData`):**

```
Promise.all([
  bookingDB.getAll(),         // → collection: 'bookings'
  submissionDB.getAll(),      // → collection: 'job_submissions'
  userMessageDB.getAll(),     // → collection: 'user_messages'
])
```

**Client-side filtering:**
- **Pending bookings:** `status === 'pending'` AND `scheduledAt >= todayStart`
- **New submissions:** `status` is `'new'`, `'reviewing'`, or `'shortlisted'`
- **Unread messages:** `status === 'sent'`
- **Today's bookings:** `scheduledAt` is today AND `status !== 'cancelled'`

**Actions:**
- Confirm booking → `bookingDB.update(id, { status: 'confirmed' })` + send confirmation email
- Cancel booking → `bookingDB.update(id, { status: 'cancelled' })`
- Confirm all pending → bulk update
- Cancel stale (>7 days old) → bulk update
- Mark message read → `userMessageDB.update(id, { status: 'read' })`
- Update submission status → `submissionDB.update(id, { status: newStatus })`
- PayPal lookup → Cloud Function `lookupPayPalOrder(bookingId)`
- PayPal capture → Cloud Function `capturePayPalOrder(bookingId)`

**Collections used:** `bookings`, `job_submissions`, `user_messages`

---

### 3.2 Tab: `documents` — Documents

**Purpose:** Upload, manage, and preview project documents. PDF text extraction and Google Drive sync.

**File:** `src/pages/Documents.tsx`

**Data fetched:**
- `firebaseDB.getAll('documents')` — all documents for the project

**Collections used:** `documents`

---

### 3.3 Tab: `analytics` — AdminAnalytics

**Purpose:** Visitor behavior analytics — page views, session flows, gateway info, quiz results.

**File:** `src/pages/AdminAnalytics.tsx`

**Data fetched:**
- `analyticsEventDB.getAll(5000)` — recent analytics events (capped at 5000)
- `analyticsSessionDB.getAll(5000)` — recent visitor sessions (capped at 5000)
- `firebaseDB.getAll('discover_quiz_results')` — quiz results

**Collections used:** `analytics_events`, `analytics_sessions`, `discover_quiz_results`

---

### 3.4 Tab: `people` — PeopleHub (★ Most Complex)

**Purpose:** Unified people management — pipeline (kanban), directory (table), duplicates, team & access, and 360° profile. This is the **default landing page** for admins.

**Files:**
- `src/pages/PeopleHub.tsx` — main hub with filters, selection, bulk actions
- `src/contexts/PeopleDataContext.tsx` — loads ALL people collections once, shares across sub-tabs
- `src/utils/peopleModel.ts` — `buildPeople(raw)` — the SINGLE canonical builder that merges all collections into `Person` objects
- `src/utils/personActions.ts` — all person actions (welcome, invite, create account, merge, delete, etc.)
- `src/components/people/PersonProfile.tsx` — 360° profile with 9 tabs
- `src/components/people/PipelineView.tsx` — kanban by funnel stage
- `src/components/people/DirectoryView.tsx` — table view
- `src/components/people/DuplicatesView.tsx` — duplicate detection & merge

**Data fetched ONCE by `PeopleDataContext` (shared across all sub-tabs):**

```
Promise.all([
  firebaseDB.getAll('systemUsers'),       // user accounts
  submissionDB.getAll(),                   // job_submissions (CVs)
  crmContactDB.getAll(),                   // crm_contacts
  studentCRMDB.getAll(),                   // student_crm
  bookingDB.getAll(),                      // bookings
  firebaseDB.getAll('contact_submissions'),// contact form
  callInvitationDB.getAll(),               // call_invitations
  portalMessageDB.getAll(true),            // portal_messages
  emailLogDB.getAll(),                     // email_logs
  analyticsEventDB.getAll(5000),           // analytics_events
  analyticsSessionDB.getAll(5000),         // analytics_sessions
  serviceOfferingDB.getAll(),              // service_offerings
])
```

**How `Person` objects are built (`buildPeople(raw)`):**

The builder merges all collections by **email address** (case-insensitive). For each unique email, it creates a `Person` with:

| Field | Source |
|---|---|
| `email` | normalized key across all collections |
| `name` | first non-empty from: systemUser.name, submission.fullName, crmContact.name, booking.name, contact.name |
| `phone` | first non-empty across all sources |
| `hasAccount` / `systemUser` / `role` / `username` | from `systemUsers` collection |
| `hasSubmission` / `submission` / `submissionStatus` | from `job_submissions` (newest one) |
| `hasCrm` / `crmContact` / `crmStatus` | from `crm_contacts` |
| `hasStudentCrm` / `studentCRM` / `studentCrmStatus` | from `student_crm` |
| `bookings[]` / `bookingCount` / `totalPaidUSD` | from `bookings` (matched on `email` AND `paypalPayerEmail`) |
| `hasContact` / `contactSubject` | from `contact_submissions` |
| `welcomeSent` | true if any `email_logs` entry has `status === 'sent'` for this email, OR `submission.welcomeSentAt` exists, OR `submission.invitedAt` exists |
| `invited` / `offerSent` | from submission fields |
| `hasCV` | from `submission.cvBase64` |
| `isEnrolled` | from `submission.status === 'enrolled'` |
| `funnelStage` | computed: `declined` → `enrolled` → `session_booked` → `cv_uploaded` → `account_created` → `welcomed` → `entered` |
| `types[]` | array of: `account`, `applicant`, `crm_lead`, `student_crm`, `booking_only`, `contact` |
| `sources[]` | array of: `signup`, `submission`, `crm`, `student_crm`, `booking`, `contact` |
| `createdAt` | earliest date across all sources |
| `lastActive` | latest from systemUser.lastActive or analytics_sessions |

**Funnel stage computation (`computeFunnelStage`):**
```
if submissionStatus === 'declined' → 'declined'
if isEnrolled → 'enrolled'
if hasBookings → 'session_booked'
if hasCV → 'cv_uploaded'
if hasAccount → 'account_created'
if welcomeSent → 'welcomed'
else → 'entered'
```

**Sub-tabs within PeopleHub:**

1. **Pipeline** — Kanban board grouped by `funnelStage` (7 columns). Drag-and-drop changes stage. Each card shows name, email, types, booking count, total paid.
2. **Directory** — Table with all people. Columns: name, email, phone, types, role, stage, bookings, total paid, created, last active. Supports search, filter, sort, selection.
3. **Duplicates** — Groups detected by `findDuplicateGroups(people)`: same phone (last 9 digits), same normalized name, or same email root (ignoring `._+-` in local part).
4. **Team & Access** — Uses `UserManagement.tsx` as a sub-tab. Shows team members (users with accounts), roles, permissions. Can create accounts, reset passwords, impersonate.

**Person Actions (all in `src/utils/personActions.ts`):**

| Action | Function | What it does |
|---|---|---|
| Send Welcome | `sendWelcome(p, actor)` | Sends career network welcome email, updates `submission.welcomeSentAt` |
| Free Consultation | `sendFreeConsultation(p, actor)` | Creates 100% off coupon, emails booking link |
| Invite to Call | `inviteToCall(p, actor)` | Creates `call_invitation` doc, updates submission status to `invited`, sends email |
| Welcome + Call Invite | `sendWelcomeWithCallInvite(p, actor)` | Coupon + call invite + account creation + welcome email (all-in-one) |
| Create Account | `createAccount(p, actor)` | Creates Firebase Auth user + `systemUsers` doc, sends setup email |
| Reset Password | `resetPassword(p, actor)` | Calls `sendBrandedPasswordReset` |
| Registration Link | `sendRegistrationLink(p, actor)` | Emails link to `/join-us?email=...` |
| Custom Email | `sendCustomEmail(p, subject, body, actor)` | Sends arbitrary email |
| Portal Message | `sendPortalMessage(p, title, body, priority, actor)` | Creates `portal_messages` doc |
| CV Feedback | `sendCVFeedback(p, feedback, actor)` | Sends CV feedback email |
| Send Offer | `sendOffer(p, offerType, actor)` | Sends offer email, updates `submission.offerSentAt` |
| Set Stage | `setStage(p, stage, actor)` | Updates submission status to match stage |
| Set Submission Status | `setSubmissionStatus(p, status, actor)` | Direct status update |
| Delete Person | `deletePerson(email, adminKey)` | Calls Cloud Function `deleteUserData` — deletes across ALL collections + Firebase Auth |
| Merge People | `mergePeople(primary, source, actor)` | Reassigns all data from source email to primary across 11 collections, deletes source account |

**Bulk actions:** `runBulk(people, action, onProgress)` — runs an action sequentially across selected people.

**CSV export:** `exportPeopleCSV(people)` — downloads CSV with name, email, phone, stage, types, role, username, account/CV/welcome/invite/offer flags, bookings, total paid, submission status, created, last active.

**Cross-page deep-linking:** Other pages call `setUser360Target(email)` then `onNavigate('people')`. PeopleHub reads it on mount and opens the profile.

**Collections used:** `systemUsers`, `job_submissions`, `crm_contacts`, `student_crm`, `bookings`, `contact_submissions`, `call_invitations`, `portal_messages`, `email_logs`, `analytics_events`, `analytics_sessions`, `service_offerings`, `communication_status`, `candidate_posts`

---

### 3.5 Tab: `communication` — CommunicationManagement

**Purpose:** Email logs, portal messages, welcome emails, SMTP tools, and email usage tracking.

**File:** `src/pages/CommunicationManagement.tsx`

**Data fetched:**
- `emailLogDB.getAll()` — all email logs (sorted by `sentAt` desc)
- `portalMessageDB.getAll(true)` — all portal messages
- `firebaseDB.getAll('systemUsers')` — for user lookups

**Features:**
- Email usage tracking (Resend free tier: 100/day, 3000/month)
- Today's sent emails list
- Portal message composer (broadcast or targeted)
- Welcome email sender
- SMTP test tools

**Collections used:** `email_logs`, `portal_messages`, `systemUsers`

---

### 3.6 Tab: `user-messages` — UserMessages

**Purpose:** Student/candidate-facing messaging — send messages to the admin team.

**File:** `src/pages/UserMessages.tsx`

**Data fetched:**
- `userMessageDB.getByUser(userEmail)` — messages for the current user

**Collections used:** `user_messages`

---

### 3.7 Tab: `paypal-payments` — PayPalPayments

**Purpose:** PayPal transaction management — search bookings, check PayPal order status, capture payments, edit bookings, send emails.

**File:** `src/pages/PayPalPayments.tsx`

**Data fetched on mount:**
```
Promise.all([
  bookingDB.getAll(),                          // all bookings
  firebaseDB.getAll('systemUsers'),            // for user lookups
])
```

**Client-side filtering:**
- By search query (name, email, PayPal order ID, booking ID, Tap charge ID, Stripe session ID)
- By status: all, paid, unpaid, completed, pending, cancelled
- By payment method: all, paypal, tap, stripe, manual, free, none
- Hidden bookings filter (`hiddenFromPayments` field)

**Actions:**
- PayPal lookup → Cloud Function `lookupPayPalOrder(bookingId)`
- PayPal capture → Cloud Function `capturePayPalOrder(bookingId)`
- Edit booking → `firebaseDB.update('bookings', id, updates)`
- Send confirmation email → `sendBookingConfirmationEmail(...)`
- Send custom email → `sendSubmissionResponseEmail(...)`
- Fetch all PayPal data → loops through all bookings with `paypalOrderId`

**Collections used:** `bookings`, `systemUsers`

---

### 3.8 Tab: `calendar` — CalendarManagement (★ Complex)

**Purpose:** Unified calendar — visual month view, bookings, availability, slot overrides, calendar blocks, slot groups, Google Calendar sync, settings.

**File:** `src/pages/CalendarManagement.tsx`

**Data fetched on mount (`loadData`):**
```
Promise.all([
  bookingDB.getAll(),           // bookings
  serviceOfferingDB.getAll(),   // service offerings (for pricing/duration)
  slotOverrideDB.getAll(),      // slot overrides (add/remove/block specific dates)
  calendarBlockDB.getAll(),     // recurring weekly blocks (e.g. business development)
  availabilityDB.getAll(),      // weekly availability slots
  slotGroupDB.getAll(),         // named slot groups with times and active days
])
```

**Also fetched:**
- `loadCalendarEvents()` → Cloud Function `listCalendarEventsViaCF(100)` — Google Calendar events
- `loadCalendarConfig()` → `getBookingCalendarConfig()` → reads `site_config/booking_settings` doc

**Calendar config fields (from `site_config/booking_settings`):**
- `bookingMaxDate` — max bookable date
- `bookingBlockStart` / `bookingBlockEnd` — date range where no bookings allowed
- `bookingAvailableDays` — array of day-of-week numbers (0=Sun, 6=Sat)
- `bookingBerlinHourStart` / `bookingBerlinHourEnd` — operating hours (Berlin timezone)
- `bookingSlotIntervalMinutes` — slot granularity (default 30)
- `bookingFixedPrice` — default price
- `bookingLimitedSlotsMode` — whether to cap slots per day
- `bookingMaxSlotsPerDay` — max slots when limited mode is on
- `bookingMinLeadHours` — minimum lead time for bookings

**Sub-tabs:** `calendar` | `bookings` | `heatmap` | `paypal`

**Slot generation logic:**
1. Start with `availableDays` from config
2. Check `slotOverrides` for date-specific additions/removals/blocks
3. Check `calendarBlocks` for recurring weekly blocks
4. Generate time slots from `berlinHourStart` to `berlinHourEnd` at `slotIntervalMin` intervals
5. Remove blocked slots
6. Remove already-booked times (from bookings + Google Calendar events)

**Actions:**
- Create booking → `bookingDB.create(...)` + Google Calendar sync + confirmation email
- Update booking status → `bookingDB.update(id, { status })` + status email + auto-create session outcome if completed
- Reschedule booking → update `scheduledAt`
- Delete booking → `bookingDB.delete(id)`
- Add/edit/delete slot override → `slotOverrideDB.create/update/delete`
- Add/edit/delete availability → `availabilityDB.create/update/delete`
- Add/edit/delete calendar block → `calendarBlockDB.create/update/delete`
- Add/edit/delete slot group → `slotGroupDB.create/update/delete`
- Save calendar settings → `saveBookingCalendarConfig(updates)` → writes to `site_config/booking_settings`

**Collections used:** `bookings`, `service_offerings`, `slot_overrides`, `calendar_blocks`, `availability`, `slot_groups`, `site_config`

**Cloud Functions:** `listCalendarEvents`, `syncBookingViaCloudFunction`, `deleteCalendarEvent`

---

### 3.9 Tab: `bookings` — BookingsManagement

**Purpose:** View and manage all client bookings with PayPal trial detection, payment details, email center, and pending analysis.

**File:** `src/pages/BookingsManagement.tsx`

**Data fetched on mount (`loadData`):**
```
Promise.all([
  bookingDB.getAll(),                          // all bookings
  serviceOfferingDB.getAll(),                  // for offering names
  firebaseDB.getAll('systemUsers'),            // for user lookups
])
```

**Views:** `all` | `upcoming` | `pending` | `trials` | `paypal`

**Client-side filtering:**
- **All:** filter by status (all, pending, confirmed, completed, cancelled)
- **Upcoming:** `scheduledAt > now` AND `status !== 'cancelled'`
- **Pending:** `status === 'pending'` (with "stuck reason" analysis)
- **Trials:** bookings with `paymentStatus === 'unpaid'` AND `priceUSD > 0`
- **PayPal:** search by email, then check PayPal order status

**Stuck reason analysis (for pending bookings):**
- Payment abandoned (>48h unpaid) — high severity
- Payment incomplete (>24h unpaid) — medium severity
- Payment pending (<24h unpaid) — low severity
- Free booking not confirmed — low severity

**Actions:**
- Update status → `bookingDB.update(id, { status })` + email + session outcome auto-create
- Delete → `bookingDB.delete(id)`
- Create booking → `bookingDB.create(...)` + Google Calendar sync + emails
- Send apology email → cancel + email
- Offer free meeting with coupon → create coupon + email + cancel
- Offer meeting (no coupon) → email + cancel
- Resend confirmation → `sendBookingConfirmationEmail(...)`
- Send reminders → `sendBookingReminders()` (24h before sessions)
- Copy reschedule link → `/reschedule?booking_id=...`
- PayPal lookup/capture → Cloud Functions
- Email center → send reconfirm/welcome/apology/custom emails

**Collections used:** `bookings`, `service_offerings`, `systemUsers`, `coupons`, `session_outcomes`

---

### 3.10 Tab: `invitations-coupons` — InvitationsCoupons

**Purpose:** Create shareable booking links, manage coupons, and send personalized invitations.

**File:** `src/pages/InvitationsCoupons.tsx`

**Data fetched:**
- `couponDB.getAll()` — all coupons
- `callInvitationDB.getAll()` — all call invitations
- `serviceOfferingDB.getAll()` — for linking coupons to offerings

**Collections used:** `coupons`, `call_invitations`, `service_offerings`

---

### 3.11 Tab: `service-offerings` — ServiceOfferingsManagement

**Purpose:** Manage outcome-based service packages and pricing (mentorship, calls, MVP pricing).

**File:** `src/pages/ServiceOfferingsManagement.tsx`

**Data fetched:**
- `serviceOfferingDB.getAll()` — all service offerings

**ServiceOffering fields:** `name`, `nameAr`, `description`, `type`, `audience`, `category`, `priceUSD`, `originalPriceUSD`, `durationMinutes`, `outcome`, `deliverables[]`, `isActive`, `featured`, `order`

**Collections used:** `service_offerings`

---

### 3.12 Tab: `session-outcomes` — OutcomeManagement

**Purpose:** Capture and publish session outcomes (proof-of-work from sessions).

**File:** `src/pages/OutcomeManagement.tsx`

**Data fetched:**
- `sessionOutcomeDB.getAll()` — all session outcomes

**Collections used:** `session_outcomes`

---

### 3.13 Tab: `price-escalation` — PriceEscalationManagement

**Purpose:** Auto-raise prices when demand fills capacity repeatedly.

**File:** `src/pages/PriceEscalationManagement.tsx`

**Data fetched:**
- `firebaseDB.getAll('price_escalation_rules')` — escalation rules
- `serviceOfferingDB.getAll()` — linked offerings
- `bookingDB.getAll()` — for demand analysis

**Collections used:** `price_escalation_rules`, `service_offerings`, `bookings`

---

### 3.14 Tab: `proposal-generator` — ProposalGenerator

**Purpose:** Generate proposals for clients from templates or service offerings.

**File:** `src/pages/ProposalGenerator.tsx`

**Data fetched:**
- `serviceOfferingDB.getAll()` — for building proposals
- `firebaseDB.getAll('proposals')` — saved proposals

**Collections used:** `proposals`, `service_offerings`

---

### 3.15 Tab: `booking-preview` — BookingPreview

**Purpose:** Preview the public booking page as seen by visitors.

**File:** `src/pages/BookingPreview.tsx`

**Data fetched:**
- `serviceOfferingDB.getActive()` — active offerings
- `getBookingCalendarConfig()` — calendar settings
- `availabilityDB.getAll()` — availability slots
- `slotOverrideDB.getAll()` — overrides
- `calendarBlockDB.getAll()` — blocks
- `slotGroupDB.getAll()` — slot groups

**Collections used:** `service_offerings`, `site_config`, `availability`, `slot_overrides`, `calendar_blocks`, `slot_groups`

---

### 3.16 Tab: `website-manager` — WebsiteManager

**Purpose:** Manage contact form submissions and career opportunities (job postings).

**File:** `src/pages/WebsiteManager.tsx`

**Data fetched:**
- `firebaseDB.getAll('contact_submissions')` — contact form messages
- `firebaseDB.getAll('career_opportunities')` — job postings

**Collections used:** `contact_submissions`, `career_opportunities`

---

### 3.17 Tab: `projects` — ProjectManagement

**Purpose:** Manage projects (name, type, description, attached tabs, status).

**File:** `src/pages/ProjectManagement.tsx`

**Data fetched:**
- `projectDB.getAll()` — all projects

**Collections used:** `projects`

---

### 3.18 Tab: `tap-config` — TapConfigManagement

**Purpose:** Manage Tap payment gateway configuration, pricing, and test payments.

**File:** `src/pages/TapConfigManagement.tsx`

**Data fetched:**
- `firebaseDB.getAll('tap_config')` — Tap configuration

**Collections used:** `tap_config`

---

### 3.19 Tab: `admin-settings` — AdminSettings

**Purpose:** Audit log viewer and system settings.

**File:** `src/pages/AdminSettings.tsx`

**Data fetched:**
- `firebaseDB.getAll('audit_logs')` — audit log entries
- `firebaseDB.getAll('paypal_config')` — PayPal configuration

**Collections used:** `audit_logs`, `paypal_config`

---

### 3.20 Tab: `link-manager` — LinkManager

**Purpose:** Manage generated shared report links, track visits, and revoke access.

**File:** `src/pages/LinkManager.tsx`

**Data fetched:**
- `firebaseDB.getAll('shared_links')` — all shared links

**Collections used:** `shared_links`

---

### 3.21 Tab: `help-guide` — HelpGuide

**Purpose:** Comprehensive help and guide for admins.

**File:** `src/pages/HelpGuide.tsx`

**Data fetched:** None (static content).

---

## 4. Cloud Functions Reference

These Cloud Functions are called from the frontend via `httpsCallable`. The iOS app should call the same functions.

| Function | Payload | Returns | Used By |
|---|---|---|---|
| `createPayPalOrder` | `{ bookingDetails }` | `{ success, url?, orderId? }` | Booking flow |
| `capturePayPalOrder` | `{ bookingId }` | `{ success, captured?, errorType? }` | Bookings, PayPal, Inbox |
| `lookupPayPalOrder` | `{ bookingId }` | `{ success, paypalOrder?, environment? }` | Bookings, PayPal, Inbox |
| `createTabbyCheckout` | `{ bookingDetails }` | `{ success, chargeId?, url? }` | Booking flow |
| `createTapTestPayment` | `{ amount, currency }` | `{ success, chargeId?, url? }` | Tap Config |
| `sendBrandedPasswordReset` | `{ email }` | `{ success }` | Auth, People |
| `sendBookingStatusEmail` | `{ email, name, status, scheduledAt, serviceName, language, bookingId }` | `{ success }` | Bookings, Calendar |
| `sendBookingConfirmationEmail` | `{ bookingId, clientName, clientEmail, serviceName, scheduledAt, durationMinutes, language }` | `{ success }` | Bookings, Calendar |
| `sendBookingReminders` | — | `{ sentCount }` | Bookings |
| `sendSubmissionResponseEmail` | `{ to, subject, html }` | `{ success }` | People, Bookings |
| `sendCVFeedbackEmail` | `{ to, name, feedback, submissionId }` | `{ success }` | People |
| `sendOfferEmail` | `{ submissionId, offerType }` | `{ success }` | People |
| `sendPasswordSetupEmail` | `{ email, name, username }` | `{ success }` | People |
| `listCalendarEvents` | `{ maxResults }` | `{ success, events[], calendarId? }` | Calendar |
| `syncBookingViaCloudFunction` | `{ title, description, startISO, endISO, attendeeEmail, attendeeName }` | `{ success, eventId? }` | Calendar, Bookings |
| `deleteCalendarEvent` | `{ eventId }` | `{ success }` | Calendar |
| `deleteUserData` | `{ email, adminKey }` | `{ deletedDocs, authDeleted }` | People (delete) |

---

## 5. Date Handling

Firestore stores dates as `{ seconds: number, nanoseconds: number }` or as Firestore Timestamps. The web app converts them with:

```typescript
function safeDate(v: any): Date | undefined {
  if (!v) return undefined;
  if (v instanceof Date) return isNaN(v.getTime()) ? undefined : v;
  if (v?.seconds) return new Date(v.seconds * 1000 + (v.nanoseconds || 0) / 1e6);
  const d = new Date(v);
  return isNaN(d.getTime()) ? undefined : d;
}
```

**For iOS:** Use `Timestamp` from Firebase Firestore SDK and convert to `Date` / `DateFormatter` accordingly.

---

## 6. Auth & Role System

### 6.1 Roles

Defined in `src/types/index.ts`:
```
type UserRole = 'admin' | 'manager' | 'advisor' | 'client' | 'student' | 'candidate' | 'graduate';
```

### 6.2 Post-Login Routing

- **admin / manager / advisor** → Admin Portal (sidebar + tabs, project selector)
- **student / candidate / graduate / client** → Student Portal (no sidebar, no project selector)

### 6.3 SystemUsers Collection

Each user has a document in `systemUsers` with:
- `id` — Firebase Auth UID
- `name`, `email`, `username`
- `role` — one of `UserRole`
- `hasPassword` — boolean
- `authProvider` — `'google'`, `'apple'`, or undefined
- `projectIds` — array of project IDs
- `permissions` — object with `canViewFinancials`, `canManageSubscriptions`, `canGenerateReports`, `canEdit`, `canDelete`, `canComment`, `canManageUsers`, `canManageProjects`
- `createdAt`, `lastActive`

### 6.4 Impersonation

Admins can impersonate users via `AuthContext.impersonate(userId)`:
1. Fetches `systemUsers/{userId}` from Firestore
2. Stores original admin user in `localStorage` as `portal_impersonating`
3. Swaps `currentUser` in context
4. All subsequent actions are logged as the impersonated user

**For iOS:** Implement similar impersonation if needed for admin debugging.

---

## 7. Key Data Models

### 7.1 Booking

```typescript
{
  id: string;
  name: string;
  email: string;
  company?: string;
  serviceOfferingId: string;
  serviceType: string;
  scheduledAt: Date;
  durationMinutes: number;
  status: 'pending' | 'confirmed' | 'completed' | 'cancelled';
  paymentStatus: 'unpaid' | 'paid';
  priceUSD: number;
  notes?: string;
  couponCode?: string;
  paypalOrderId?: string;
  paypalPayerEmail?: string;
  tapChargeId?: string;
  stripeSessionId?: string;
  paymentMethod?: 'paypal' | 'tap' | 'stripe' | 'manual' | 'free';
  userId?: string;
  createdAt: Date;
  sessionLink?: string;
  hiddenFromPayments?: boolean;
}
```

### 7.2 JobSubmission

```typescript
{
  id: string;
  fullName: string;
  email: string;
  phone?: string;
  role: string;
  message?: string;
  cvFileName: string;
  cvFileType: string;
  cvFileSize: number;
  cvBase64: string;
  status: 'new' | 'reviewing' | 'shortlisted' | 'archived' | 'invited' | 'preparing' | 'interested' | 'declined' | 'survey_completed' | 'enrolled';
  submittedAt: Date;
  welcomeSentAt?: Date;
  welcomeSentBy?: string;
  invitedAt?: Date;
  invitedBy?: string;
  offerSentAt?: Date;
  offerType?: 'coffee_time' | 'membership' | 'course_access' | 'full_funnel';
  isGraduate?: boolean;
  jobType?: JobType | JobType[];
  careerId?: string;
  cvHash?: string;
  cvAnalysis?: { skills[], strongTopics[], gapTopics[], suggestedMilestones[], summary?, analyzedAt: Date, model? };
  deviceInfo?: { userAgent, platform, browser, os, deviceType, screenResolution, language, timezone, referrer, pageUrl, ipAddress? };
}
```

### 7.3 ServiceOffering

```typescript
{
  id: string;
  name: string;
  nameAr?: string;
  description: string;
  descriptionAr?: string;
  type: string;  // e.g. 'mentorship_coffee', 'mentorship_1x6', etc.
  audience: 'student' | 'graduate' | 'company' | 'robotics' | 'learner' | 'candidate';
  category: string;
  priceUSD: number;
  originalPriceUSD?: number;
  durationMinutes: number;
  outcome: string;
  outcomeAr?: string;
  deliverables: string[];
  deliverablesAr?: string[];
  duration: string;
  commitment: string;
  pricePer: string;
  featured: boolean;
  order: number;
  isActive: boolean;
  createdAt: Date;
}
```

### 7.4 SlotOverride

```typescript
{
  id: string;
  type: 'add' | 'remove' | 'block';
  date: string;       // YYYY-MM-DD
  endDate?: string;   // for 'remove' type (date range)
  startTime: string;  // HH:MM
  endTime: string;    // HH:MM
  label?: string;
  createdAt: Date;
}
```

### 7.5 CalendarBlock

```typescript
{
  id: string;
  dayOfWeek: number;  // 0-6
  startTime: string;  // HH:MM
  endTime: string;    // HH:MM
  label: string;
  labelAr?: string;
  isProtected: boolean;
  createdAt: Date;
}
```

### 7.6 Availability

```typescript
{
  id: string;
  dayOfWeek: number;  // 0-6
  startTime: string;  // HH:MM
  endTime: string;    // HH:MM
  bufferMinutes: number;
  maxBookingsPerDay: number;
  priceTier: 'standard' | 'premium' | 'off_peak';
  createdAt: Date;
}
```

### 7.7 Coupon

```typescript
{
  id: string;
  code: string;
  discountType: 'free' | 'percent' | 'fixed';
  discountValue: number;
  maxUses: number;
  usedCount: number;
  isActive: boolean;
  notes?: string;
  createdAt: Date;
}
```

---

## 8. iOS Implementation Notes

### 8.1 Data Fetching Strategy

1. **Mirror the web's `xxxDB` pattern:** Create Swift managers for each collection that call Firestore directly
2. **Normalize dates:** Convert Firestore `Timestamp` to `Date` in Swift
3. **Case-insensitive email matching:** Always lowercase + trim emails when comparing
4. **People builder:** Replicate `buildPeople(raw)` in Swift — it's the single source of truth for merging collections by email

### 8.2 Real-time Updates

The web app does NOT use Firestore real-time listeners (snapshot listeners). It uses manual `loadData()` calls after mutations. For iOS, you could:
- **Option A (simple):** Replicate the manual refresh pattern
- **Option B (better):** Use Firestore snapshot listeners for real-time updates — this would be an improvement over web

### 8.3 Offline Support

The web app has a localStorage fallback when Firebase is not configured. iOS should use Firestore's built-in offline persistence.

### 8.4 Bilingual Support

All labels have English and Arabic variants. The web uses `useLanguage()` context. iOS should use a similar locale system with RTL support for Arabic.

### 8.5 Project Type Filtering

When the user selects the "Website and Tools" project, filter tabs by `projectTypes.includes('website')`. The project selector is in the sidebar.

---

## 9. File Reference

| File | Purpose |
|---|---|
| `src/utils/firebase.ts` | Firebase init, `firebaseDB` CRUD wrapper, `firebaseAuth` |
| `src/utils/database.ts` | All collection-specific DB wrappers (bookingDB, submissionDB, etc.) |
| `src/utils/peopleModel.ts` | `Person` type, `buildPeople()`, `computeFunnelStage()`, `findDuplicateGroups()` |
| `src/utils/personActions.ts` | All person actions (welcome, invite, merge, delete, etc.) |
| `src/utils/emailService.ts` | Email sending functions + PayPal Cloud Function calls |
| `src/utils/googleCalendar.ts` | Google Calendar sync functions |
| `src/tabs/tabRegistry.ts` | Tab registration system + `getTabsForProject()` |
| `src/tabs/index.ts` | Imports all active tab files (triggers registration) |
| `src/tabs/*.tsx` | Individual tab registration files |
| `src/App.tsx` | Main portal rendering, tab selection, auth routing |
| `src/contexts/PeopleDataContext.tsx` | Shared people data provider (loads all collections once) |
| `src/contexts/AuthContext.tsx` | Auth state, impersonation, login/logout |
| `src/pages/InboxDashboard.tsx` | Inbox page |
| `src/pages/PeopleHub.tsx` | People hub (pipeline + directory + duplicates + team) |
| `src/pages/CalendarManagement.tsx` | Calendar management |
| `src/pages/BookingsManagement.tsx` | Bookings management |
| `src/pages/PayPalPayments.tsx` | PayPal payments |
| `src/pages/CommunicationManagement.tsx` | Communication management |
| `src/pages/WebsiteManager.tsx` | Website manager |
| `src/types/index.ts` | TypeScript type definitions |

---

Reply here with any questions.
