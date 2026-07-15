# Web → iOS: Full Sync Briefing — July 11, 2026

**From:** Web team
**To:** iOS team
**Date:** July 11, 2026
**Re:** Everything you need to sync — projects, Firestore, cloud functions, auth, calendar, payments

---

## 1. Project Consolidation — Exactly 3 Projects

We consolidated all projects (there were 6-7) into exactly **3 canonical projects**.

### The 3 Projects

| # | Name | Arabic Name | Firestore `type` | Purpose |
|---|------|-------------|-------------------|---------|
| 1 | **ADY** | `آدي` | `"accounting"` | Financial planning, receipts, transactions, periods, stores, accounting |
| 2 | **Riad Shannak** | `رياض شناك` | `"real_estate"` | Real estate: buildings, units, tenants, rent tracking |
| 3 | **Website and Tools** | `الموقع والأدوات` | `"website"` | Job submissions, CV prep, courses, bookings, consultations, proposals, CRM, website content |

### Seed Logic

On web app load (`initializeDatabase`):
1. Fetch all `projects` from Firestore
2. For each canonical type — if it exists, rename to canonical name; if not, create it
3. Old projects are **hidden**, not deleted

### Project Filtering

Only projects named `ADY`, `Riad Shannak`, `Website and Tools` are shown (case-insensitive).

### Old → New Type Mapping

| Old type | New type |
|----------|----------|
| `internal` | all 3 (admin tabs) |
| `education` | `website` |
| `consulting` | `website` |
| `client_interface` | `website` |

### Default Tabs per Project

**ADY:** `dashboard`, `documents`, `timeline`, `finance`, `transactions`, `budget`, `reports`, `settings`

**Riad Shannak:** `dashboard`, `documents`, `riad-portal`, `buildings`, `reports`, `settings`

**Website and Tools:** `dashboard`, `submissions`, `student-portal`, `student-overview`, `course-management`, `membership-management`, `cv-preparation`, `video-hub`, `latex-compiler`, `crm-contacts`, `email-inbox`, `service-offerings`, `proposal-generator`, `availability`, `bookings`, `tap-config`, `website-manager`, `settings`

### What iOS Should Do

1. Use the same 3 project names and `type` values
2. Stop using old types (`internal`, `education`, `consulting`, `client_interface`)
3. Filter project list to only the 3 canonical names

---

## 2. Firestore Collections — Complete Reference

All in project `robotics-website-5593f`. Web reads/writes via Firebase JS SDK.

### Core Collections

| Collection | Type | Notes |
|------------|------|-------|
| `projects` | Project | Both |
| `project_members` | ProjectMember | Both |
| `ady_transactions` | Transaction | Web primary; also reads `transactions` as fallback (merge by ID, most recent wins) |
| `transactions` | Transaction | iOS writes here; web reads as secondary |
| `ady_periods` | Period | Web primary; also reads `periods` as fallback |
| `periods` | Period | iOS writes here; web reads as secondary |
| `ady_installments` | Installment | Both |
| `ady_stores` | Store | Both |
| `subscriptions` | Subscription | Both |
| `documents` | PortalDocument | Both |
| `comments` | Comment | Both |
| `buildings` | Building | Both |
| `ady_crm` | CRMLead | Both |
| `timeline` | TimelineEntry | Both |
| `audit_log` | AuditEntry | Web (iOS can write) |

### Website & Student Collections

| Collection | Type |
|------------|------|
| `job_submissions` | JobSubmission |
| `contact_submissions` | ContactSubmission |
| `career_opportunities` | CareerOpportunity |
| `service_offerings` | ServiceOffering |
| `bookings` | Booking |
| `availability` | Availability |
| `tap_config` | — |
| `crm_contacts` | CRMContact |
| `student_crm` | StudentCRMContact |
| `email_responses` | EmailResponse |
| `time_slot_config` | TimeSlotConfig |
| `time_slot_bookings` | TimeSlotBooking |
| `call_invitations` | CallInvitation |
| `portal_messages` | PortalMessage |
| `site_config` | SiteConfig |
| `quizzes` | Quiz |
| `widget_configs` | ProjectWidgetConfig |
| `latex_templates` | LatexTemplateDoc |

### Student DB Collections

| Collection | Type |
|------------|------|
| `courses` | Course |
| `lectures` | Lecture |
| `enrollments` | Enrollment |
| `placement_tests` | PlacementTest |
| `placement_test_results` | PlacementTestResult |
| `video_tutorials` | VideoTutorial |
| `student_invitations` | StudentInvitation |
| `membership_plans` | MembershipPlan |
| `student_memberships` | StudentMembership |
| `cv_templates` | CVTemplate |
| `prepared_cvs` | PreparedCV |
| `video_assignments` | StudentVideoAssignment |

### Special Collections

| Collection | Purpose |
|------------|---------|
| `systemUsers` | User profiles (keyed by Firebase Auth UID) |
| `venture_reports` | Shareable venture reports (token-based public read) |
| `agent_log` | AI agent log (single doc, id=`main`) |

### Transaction Merge Logic (Important)

Web reads **both** `ady_transactions` AND `transactions`, merging by ID (most recent `updatedAt`/`createdAt` wins). Same for `ady_periods` AND `periods`. iOS can write to `transactions`/`periods` — web will see them.

### Date Handling

Firestore Timestamps can arrive as `{ seconds, nanoseconds }`, `{ _seconds, _nanoseconds }`, ISO string, epoch number, or `Date`. Web uses `safeDate()` to handle all formats — iOS should do the same.

---

## 3. Authentication

### Firebase Auth (Primary)

- Email/password + Google + Apple Sign-In
- User profiles in `systemUsers` collection, keyed by Firebase Auth UID
- Secondary auth instance used for admin-creating users without disrupting admin session
- Apple: `OAuthProvider('apple.com')` with `idToken`
- Google: `GoogleAuthProvider.credential(idToken)`

### Legacy Admin Fallback

- Username: `shannak`, bcrypt-hashed password
- Fallback hash: `$2b$12$zAUUD7WXC.fit8pu4.QOR.X1UU/d.cHy1igd0mEr8dxTixf87ldX.`
- Admin whitelist: `mohammedshannak@gmail.com`

### Login Access Rules

1. Admin emails (hardcoded whitelist) → always allowed
2. Has `systemUsers` profile → allowed
3. Has `job_submissions` doc with matching email → allowed

### User Roles

`'admin' | 'manager' | 'student' | 'client' | 'advisor' | 'candidate'`

### User Permissions (14 fields)

`canViewDashboard`, `canViewFinancials`, `canViewProjectManagement`, `canViewCRM`, `canViewContracts`, `canViewTransactions`, `canViewReports`, `canManageSubscriptions`, `canGenerateReports`, `canEdit`, `canDelete`, `canComment`, `canManageUsers`, `canManageProjects`

---

## 4. Cloud Functions — Complete Reference

All deployed in `us-central1`. Callable functions require Firebase Auth.

### Callable (`onCall` — require auth)

| Function | Purpose | Secrets |
|----------|---------|---------|
| `sendCVFeedback` | Send CV feedback email to student | `SMTP_PASS`, `SMTP_USER` |
| `sendInvitationEmail` | Send CV invitation to student | `SMTP_PASS`, `SMTP_USER` |
| `sendSubmissionResponse` | Send branded HTML email to applicant | `SMTP_PASS` |
| `setupAdmin` | Create/repair admin account | `ADMIN_SETUP_KEY` |
| `cleanupSystemUsers` | Remove duplicate systemUsers | `ADMIN_SETUP_KEY` |
| `extractCVData` | Extract structured CV data via Gemini | `GEMINI_API_KEY` |
| `sendContactReply` | Reply to contact form submission | `SMTP_PASS`, `SMTP_USER` |
| `createTapCheckout` | Create Tap payment session | `TAP_SECRET_KEY` |
| `sendOfferEmail` | Send offer email to candidate | `SMTP_PASS`, `SMTP_USER` |
| `sendBookingStatus` | Send booking status update email | `SMTP_PASS`, `SMTP_USER` |
| `sendQuotation` | Send quotation email | `SMTP_PASS`, `SMTP_USER` |
| `sendBookingConfirmation` | Send booking confirmation email | `SMTP_PASS`, `SMTP_USER` |
| `sendBookingReminder` | Send booking reminder email | `SMTP_PASS`, `SMTP_USER` |
| `syncBookingToCalendar` | Sync booking to Google Calendar | `GOOGLE_SERVICE_ACCOUNT_EMAIL`, `GOOGLE_SERVICE_ACCOUNT_PRIVATE_KEY`, `GOOGLE_CALENDAR_ID` |

### HTTP (`onRequest` — public or webhook)

| Function | Purpose |
|----------|---------|
| `captureEmailResponse` | Public endpoint for student email responses |
| `processInboundEmail` | Resend inbound email webhook |
| `tapWebhook` | Tap payment webhook (CHARGE_SUCCEEDED) |
| `captureQuotationResponse` | Public endpoint for quotation responses |

### Email Config

- **From:** `"Robotics Sciences" <info@the-rs.com>`
- **Reply-To:** `info@the-rs.com`
- **Site URL:** `https://robotics-website-5593f.web.app`
- Anti-spam headers: `X-Mailer`, `X-Priority`, `List-Unsubscribe`, `Feedback-ID`

### Gemini Model

Using `gemini-2.5-flash` (updated from `gemini-2.0-flash` which shut down June 1, 2026).

---

## 5. Calendar Sync

### `syncBookingToCalendar` (Callable)

**Request:**
```typescript
{
  title: string;
  description: string;
  startISO: string;      // ISO-8601
  endISO: string;        // ISO-8601
  attendeeEmail?: string;
  attendeeName?: string;
  bookingId?: string;
}
```

**Returns:** `{ success: true, eventId: string, htmlLink: string }`

- Google Service Account JWT auth
- Calendar timezone: `Asia/Amman`
- iOS can call directly via Firebase Functions SDK

---

## 6. Tap Payments (Jordan)

### `createTapCheckout` (Callable)

**Request:**
```typescript
{
  serviceOfferingId: string;
  firstName: string;
  lastName: string;
  email: string;
  scheduledAt: string;   // ISO-8601
}
```

**Flow:**
1. Fetch `service_offerings` doc for pricing
2. Create booking in `bookings` with status `pending`, paymentStatus `unpaid`
3. Call Tap API (`https://api.tap.company/v2/charges`)
4. Update booking with `tapChargeId`
5. Return `{ success, chargeId, url }` — redirect to `url`

**Tap Webhook:** Updates booking to `confirmed`/`paid`, sends confirmation email.

**Redirect URLs:**
```
post.url    = {SITE_URL}/book/confirm?booking_id={bookingId}
redirect.url = {SITE_URL}/book/confirm?booking_id={bookingId}&tap_id=PLACEHOLDER
```

---

## 7. Service Offerings (Seed Data)

| # | Name | Type | Audience | Price | Duration |
|---|------|------|----------|-------|----------|
| 1 | Quick Coffee Chat | `mentorship_coffee` | learner | $7.50 | 15 min |
| 2 | Mentorship 1x/month 6mo | `mentorship_1x6` | learner | $150 | 6×60 min |
| 3 | Mentorship 2x/month 3mo | `mentorship_2x3` | learner | $180 | 6×60 min |
| 4 | Mentorship 4x/month | `mentorship_4x` | learner | $120 | 4×60 min |
| 5 | Discovery Call 30 min | `discovery_call_30` | robotics | $35 | 30 min |
| 6 | Discovery Call 60 min | `discovery_call_60` | robotics | $45 | 60 min |
| 7 | Robotics MVP | `robotics_mvp` | robotics | $700 | Custom |
| 8 | System Assessment | `company_assessment` | company | $0 | 60 min |

**Audience values:** `'student' | 'graduate' | 'company' | 'robotics' | 'learner'`
**Category values:** `'career' | 'training' | 'advisory' | 'technical' | 'robotics' | 'mentorship'`

---

## 8. Availability (Seed Data)

- Sun–Thu: 10:00–18:00, 15-min buffer, max 5/day
- Fri: 10:00–16:00, 15-min buffer, max 3/day
- Sat: closed

---

## 9. Membership Plans (Seed Data)

| Plan | Price | Cycle |
|------|-------|-------|
| Coffee Time | $7 | one-time |
| Curriculum Builder | $25/mo | monthly |
| Course Access | $12/mo | monthly |

---

## 10. Site Config (`site_config`)

Shared config doc (single doc, keyed by config ID):

```typescript
{
  photoUrl?: string;           // Discover page photo
  photoScale?: number;
  photoOffsetX?: number;
  photoOffsetY?: number;
  calendarConnected?: boolean;
  showNews?: boolean;          // Feature flags
  showProjects?: boolean;
  showCourses?: boolean;
  showMembership?: boolean;
  showTutorials?: boolean;
  roles?: any[];
  passwordHash?: string;       // Legacy admin
}
```

iOS can read/write this doc to sync feature flags and calendar connection state.

---

## 11. Shared Budget Dashboard

- Web renders shared budget client-side (no Express endpoint needed)
- `SharedBudgetDashboard` component fetches data via `fetchSharedBudgetData` using a token in the URL
- Token queries Firestore `sharedViews` collection (public read)
- Currency conversion: USD → JOD at 0.707, EUR → JOD at 0.78
- Expense detection: `cost`/`bill`/`freelance`/`salary` = expense; `revenue`/`payment`/`check` = income

---

## 12. Venture Reports

- `venture_reports` collection — token-based public read
- `VentureReportPage` fetches by `shareToken` field
- Also loads `timeline` entries linked by `project_id`
- iOS can write venture reports; web renders them

---

## 13. Student Onboarding

- `StudentOnboardingPage` fetches invitation from `student_invitations` by `inviteId`
- Loads CV template from `cv_templates`
- Updates invitation status through the onboarding flow
- iOS can create invitations; web handles the onboarding UI

---

## 14. Recent Changes (Since Last Sync)

- **Gemini model** updated to `gemini-2.5-flash` (2.0 shut down June 1, 2026)
- **Web portal tabs** consolidated from 37 → 20
- **Project types** consolidated from 7 → 3
- **Transaction merge** — web now reads both `ady_transactions` and `transactions` collections
- **Email response system** — `captureEmailResponse` HTTP function + `email_responses` collection
- **CV extraction** — `extractCVData` callable function using Gemini
- **Contact reply** — `sendContactReply` callable function
- **Tap payments** — `createTapCheckout` + `tapWebhook` for Jordan payments
- **Quotation system** — `sendQuotation` + `captureQuotationResponse`
- **Booking confirmation** — `sendBookingConfirmation` + `sendBookingReminder`
- **Calendar sync** — `syncBookingToCalendar` using Service Account JWT
- **Portal messages** — `portal_messages` collection for site-wide + targeted messaging
- **Time slot config** — tier-based student scheduling with `time_slot_config` + `time_slot_bookings`
- **Call invitations** — `call_invitations` collection for student call invites

---

## Questions for iOS

1. Do you have projects in Firestore with types other than `accounting`/`real_estate`/`website`?
2. Should we write a migration script to reassign old project data to canonical projects?
3. Are you using the `type` field to filter tabs/screens? If so, update to use only the 3 types.
4. Are you reading from `ady_transactions` or `transactions`? Web reads both — make sure your reads are consistent.
5. Are you calling any Cloud Functions directly? All callable functions require Firebase Auth.
