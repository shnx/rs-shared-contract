# Web Update — iOS Team — July 30, 2026 (People Refactor)

> **From:** Web Team
> **To:** iOS Team
> **Subject:** People area consolidation, batched Firestore fetching, and new architecture — what changed and what iOS needs to know

---

## 1. People Area — Consolidated from 5 tabs → 1 tab + Communication

### What was deleted

- `FunnelManager.tsx` — replaced by PeopleHub ▸ Pipeline sub-tab
- `User360.tsx` — replaced by PeopleHub ▸ Person Profile panel
- `PeopleManagement.tsx` — replaced by PeopleHub ▸ Directory sub-tab
- `funnelManagerTab.tsx`, `user360Tab.tsx`, `userManagementTab.tsx` — tab registrations removed
- ~750 lines of dead code removed from `UserManagement.tsx` (duplicated "Candidates & Submissions" view)

### New architecture (single source of truth)

| File | Purpose |
|---|---|
| `src/utils/peopleModel.ts` | Canonical `Person` type + `buildPeople(raw)` builder + `computeFunnelStage()` + `findDuplicateGroups()`. **ONLY place** a person is assembled from raw collections. |
| `src/contexts/PeopleDataContext.tsx` | React context that loads ALL people collections ONCE and shares across sub-tabs. Exposes `people`, `peopleByEmail`, `duplicates`, `raw`, `reload()`, `removeLocal()`. |
| `src/utils/personActions.ts` | ALL person actions centralized as pure async fns returning `ActionResult {ok, msg}`. |
| `src/pages/PeopleHub.tsx` | Main hub — owns filters, selection, bulk actions. Sub-tabs: Pipeline \| Directory \| Duplicates \| Team & Access. |
| `src/components/people/PersonProfile.tsx` | Unified 360° profile with 9 tabs and all actions. |
| `src/components/people/PipelineView.tsx` | Kanban board by funnel stage. |
| `src/components/people/DirectoryView.tsx` | Table view with search, filter, sort. |
| `src/components/people/DuplicatesView.tsx` | Duplicate detection & merge UI. |
| `src/components/people/stageMeta.ts` | Stage metadata (colors, labels, icons). |

### PeopleHub sub-tabs

1. **Pipeline** — Kanban board grouped by `funnelStage` (7 columns: entered → welcomed → account_created → cv_uploaded → session_booked → enrolled → declined). Each card shows name, email, types, booking count, total paid.
2. **Directory** — Table with all people. Columns: name, email, phone, types, role, stage, bookings, total paid, created, last active. Supports search, filter, sort, selection.
3. **Duplicates** — Groups detected by same phone (last 9 digits), same normalized name, or similar email root (ignoring `._+-` in local part). Merge action reassigns all data across 11 collections.
4. **Team & Access** — Uses `UserManagement.tsx` as sub-tab. Shows team members, roles, permissions. Can create accounts, reset passwords, impersonate.

### Funnel stage computation

```
if submissionStatus === 'declined' → 'declined'
if isEnrolled → 'enrolled'
if hasBookings → 'session_booked'
if hasCV → 'cv_uploaded'
if hasAccount → 'account_created'
if welcomeSent → 'welcomed'
else → 'entered'
```

### Person actions (all in `personActions.ts`)

| Action | Function | What it does |
|---|---|---|
| Send Welcome | `sendWelcome(p, actor)` | Sends career network welcome email, updates `submission.welcomeSentAt` |
| Free Consultation | `sendFreeConsultation(p, actor)` | Creates 100% off coupon, emails booking link |
| Invite to Call | `inviteToCall(p, actor)` | Creates `call_invitation` doc, updates submission status to `invited`, sends email |
| Welcome + Call Invite | `sendWelcomeWithCallInvite(p, actor)` | Coupon + call invite + account creation + welcome email |
| Create Account | `createAccount(p, actor)` | Creates Firebase Auth user + `systemUsers` doc, sends setup email |
| Reset Password | `resetPassword(p, actor)` | Calls `sendBrandedPasswordReset` |
| Registration Link | `sendRegistrationLink(p, actor)` | Emails link to `/join-us?email=...` |
| Custom Email | `sendCustomEmail(p, subject, body, actor)` | Sends arbitrary email |
| Portal Message | `sendPortalMessage(p, title, body, priority, actor)` | Creates `portal_messages` doc |
| CV Feedback | `sendCVFeedback(p, feedback, actor)` | Sends CV feedback email |
| Send Offer | `sendOffer(p, offerType, actor)` | Sends offer email, updates `submission.offerSentAt` |
| Set Stage | `setStage(p, stage, actor)` | Updates submission status |
| Delete Person | `deletePerson(email, adminKey)` | Calls CF `deleteUserData` — deletes across ALL collections + Firebase Auth |
| Merge People | `mergePeople(primary, source, actor)` | Reassigns all data from source email to primary across 11 collections, deletes source account |
| Bulk Actions | `runBulk(people, action, onProgress)` | Runs an action sequentially across selected people |
| CSV Export | `exportPeopleCSV(people)` | Downloads CSV with all person fields |

### Cross-page deep-linking

Other pages (BookingsManagement, PayPalPayments) call `setUser360Target(email)` then `onNavigate('people')`. PeopleHub reads it on mount and opens the profile.

### Default landing page

Changed from `funnel-manager` to `people` in App.tsx.

### iOS action

If iOS has separate Funnel Manager, User 360, and People Management screens, consolidate them into a single **People** tab with sub-tabs matching the web. The `buildPeople()` function is the single source of truth — replicate it in Swift to merge all collections by email.

---

## 2. Batched Firestore Fetching — New `batchedGetAll`

### What changed

Added `firebaseDB.batchedGetAll<T>(collectionName, batchSize=500)` to `src/utils/firebase.ts`. This method fetches documents in paginated chunks using Firestore cursor pagination (`limit` + `startAfter`), instead of downloading an entire collection in one `getDocs` call.

### Why

Large collections (bookings, email_logs, analytics_events) were hitting Firestore's default response size limits and causing slow loads. Batched fetching:
- Reduces per-query memory pressure
- Avoids Firestore timeout on very large collections
- Allows progressive loading if needed in the future

### How it works

```typescript
async batchedGetAll<T>(collectionName: string, batchSize = 500): Promise<T[]> {
  // Fetches first 500 docs, then uses startAfter(lastDoc) to fetch next 500, etc.
  // Concatenates all results into a single array.
  // Falls back to localStorage if Firebase is not configured.
}
```

### Which DB wrappers were updated

All 10 wrappers used by `PeopleDataContext` now use `batchedGetAll` instead of `getAll`:

- `submissionDB.getAll()` → `firebaseDB.batchedGetAll('job_submissions')`
- `bookingDB.getAll()` → `firebaseDB.batchedGetAll('bookings')`
- `crmContactDB.getAll()` → `firebaseDB.batchedGetAll('crm_contacts')`
- `studentCRMDB.getAll()` → `firebaseDB.batchedGetAll('student_crm')`
- `callInvitationDB.getAll()` → `firebaseDB.batchedGetAll('call_invitations')`
- `portalMessageDB.getAll()` → `firebaseDB.batchedGetAll('portal_messages')`
- `emailLogDB.getAll()` → `firebaseDB.batchedGetAll('email_logs')`
- `analyticsEventDB.getAll()` → `firebaseDB.batchedGetAll('analytics_events')`
- `analyticsSessionDB.getAll()` → `firebaseDB.batchedGetAll('analytics_sessions')`
- `serviceOfferingDB.getAll()` → `firebaseDB.batchedGetAll('service_offerings')`

### `fetchAll` refactored — 3 waves instead of 1 big `Promise.all`

Previously: 12 simultaneous Firestore queries in a single `Promise.all`.

Now: 3 sequential waves of 4 concurrent requests each:

```
Wave 1: systemUsers, submissions, crmContacts, studentCRMs
Wave 2: bookings, contactSubmissions, callInvitations, portalMessages
Wave 3: emailLogs, analyticsEvents, analyticsSessions, offerings
```

This reduces peak concurrent Firestore connections from 12 to 4.

### iOS action

If iOS is fetching these same collections, consider:
1. **Use Firestore's `getDocuments(source: .cache)` with `limit` + `startAfter`** for the same batched approach
2. **Or use Firestore snapshot listeners** for real-time updates (an improvement over web's manual refresh)
3. **The data shape is identical** — `batchedGetAll` returns the same `T[]` as `getAll`, just fetched in chunks

---

## 3. Collections Used by PeopleDataContext

The PeopleDataContext loads these 12 collections on mount:

| Collection | Variable | Purpose |
|---|---|---|
| `systemUsers` | `systemUsers` | User accounts, roles, auth provider |
| `job_submissions` | `submissions` | CV submissions, application status |
| `crm_contacts` | `crmContacts` | CRM contacts (client-facing) |
| `student_crm` | `studentCRMs` | Student CRM (separate from client CRM) |
| `bookings` | `bookings` | All bookings (matched on email + paypalPayerEmail) |
| `contact_submissions` | `contactSubmissions` | Contact form submissions |
| `call_invitations` | `callInvitations` | Call invitations sent to students |
| `portal_messages` | `portalMessages` | Portal messages (broadcast + targeted) |
| `email_logs` | `emailLogs` | Email send logs (used for welcome detection) |
| `analytics_events` | `analyticsEvents` | Analytics events (capped at 5000) |
| `analytics_sessions` | `analyticsSessions` | Visitor sessions (capped at 5000) |
| `service_offerings` | `offerings` | Service offerings (for booking context) |

Additionally, `mergePeople` touches these collections:
- `communication_status`, `user_messages`, `candidate_posts`, `email_responses`

---

## 4. Complete Portal Tab List (Updated)

The active tabs for the `website` project type (21 tabs total):

### Dashboard
1. `inbox` — InboxDashboard
2. `documents` — Documents
3. `analytics` — AdminAnalytics

### People
4. `people` — PeopleHub (★ NEW — replaces funnel-manager, user-360, user-management, submissions)
5. `communication` — CommunicationManagement
6. `user-messages` — UserMessages

### Finance
7. `paypal-payments` — PayPalPayments

### Bookings & Services
8. `calendar` — CalendarManagement
9. `bookings` — BookingsManagement
10. `invitations-coupons` — InvitationsCoupons
11. `service-offerings` — ServiceOfferingsManagement
12. `session-outcomes` — OutcomeManagement
13. `price-escalation` — PriceEscalationManagement
14. `proposal-generator` — ProposalGenerator
15. `booking-preview` — BookingPreview
16. `website-manager` — WebsiteManager

### Settings
17. `projects` — ProjectManagement
18. `tap-config` — TapConfigManagement
19. `admin-settings` — AdminSettings
20. `link-manager` — LinkManager
21. `help-guide` — HelpGuide

### Deleted tabs (no longer in portal)
- `funnel-manager` → merged into `people`
- `user-360` → merged into `people`
- `user-management` → merged into `people` (Team & Access sub-tab)
- `submissions` → merged into `people`
- `crm-contacts` → merged into `people`
- `email-inbox` → merged into `communication`
- `launchpad` → removed
- `dashboard` → removed (inbox replaces)
- `four-numbers` → removed

---

## 5. Full Data Sync Spec Available

We've also published `IOS_PORTAL_DATA_SYNC_SPEC.md` in the shared docs — a comprehensive 38KB document covering:

- All 21 active tabs with their data sources and fetching patterns
- Complete Firestore collection reference (30+ collections)
- All Cloud Functions with payloads and responses
- Key data models (Booking, JobSubmission, ServiceOffering, SlotOverride, CalendarBlock, Availability, Coupon)
- Auth & role system
- Date handling
- File reference map
- iOS implementation notes

**Read `IOS_PORTAL_DATA_SYNC_SPEC.md` for the full technical spec.**

---

## Summary of iOS action items

| Priority | Item |
|---|---|
| P1 | Consolidate separate Funnel/People/User360 screens into single People tab with sub-tabs |
| P1 | Replicate `buildPeople()` in Swift — merge all collections by email |
| P2 | Consider batched Firestore fetching (limit + startAfter) for large collections |
| P2 | Implement person actions (welcome, invite, merge, delete) matching `personActions.ts` |
| P3 | Consider Firestore snapshot listeners instead of manual refresh (improvement over web) |

---

Reply here with any questions.
