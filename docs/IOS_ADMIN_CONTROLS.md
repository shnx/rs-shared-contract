# iOS Admin — Full Web Controls Handoff

**Date:** July 2026
**Project:** Robotics Sciences website (`/Users/mohammadshannak/Documents/RS_website`)
**Firebase project:** `robotics-website-5593f`
**Cloud Functions region:** `us-central1`
**This doc is for:** iOS developers who want to build admin views/controls that mirror the web portal.

---

## 1. What this doc covers

This is a **detailed, not overview** guide of every admin-controllable concept in the web portal. For each concept it lists:

- The exact Firestore **collection ID** to read/write.
- The **schema**, expected field values, and validation rules.
- The **web controls** the admin already has.
- A concrete **iOS view/control suggestion** with the best UX for a native admin app.
- Any **Cloud Functions / callable endpoints** that should be used instead of writing raw Firestore operations.

It is grouped from **most important for iOS** (P1) to **nice to have** (P3). The iOS app should share the same Firestore database and the same `systemUsers` roles/permissions as the web.

---

## 2. Firebase setup for iOS

```swift
import Firebase

FirebaseApp.configure(options: FirebaseOptions(googleAppID: "...", gcmSenderID: "..."))
// Firestore
let db = Firestore.firestore()
db.settings.isPersistenceEnabled = true
// Functions
let functions = Functions.functions(region: "us-central1")
// Auth
Auth.auth().signIn(withEmail: email, password: password)
```

Key shared IDs:

| Platform | Collection / Resource | Value |
|---|---|---|
| Project ID | Firebase console | `robotics-website-5593f` |
| Functions region | Cloud Functions | `us-central1` |
| User profiles | Firestore | `systemUsers` |
| Auth roles | `systemUsers/{uid}` → `role` | `admin`, `manager`, `advisor`, `student`, `client`, `candidate` |

> iOS should mirror the web `User.permissions` object when deciding which UI to show. The web reads `systemUsers/{uid}` after sign-in and caches `permissions`.

---

## 3. Common data conventions

### Dates

Firestore stores most dates as ISO strings or `Timestamp`. iOS can read either and should convert to `Date`/`Timestamp`. The helper convention in the web is:

- `createdAt`, `updatedAt`, `scheduledAt`, `expiresAt`, `submittedAt`, etc. are **dates**.
- `dayOfWeek` is `0 = Sunday` through `6 = Saturday`.
- `startTime` / `endTime` are `HH:mm` strings (e.g. `"10:00"`).
- `startHour` / `endHour` are integers `0`–`23`.

### Currency / prices

- **Consultation bookings (Tap):** `USD` major unit (e.g. `4.5` = $4.50). The `createTapCheckout` function multiplies to `450` for Tap internally, but the admin writes `4.5`.
- **Time-slot based student bookings:** `JOD` (Jordanian Dinar), stored as `priceJOD`.
- **Courses / memberships:** `USD` stored in `price`.

### Status enums

- `Booking.status`: `pending` | `confirmed` | `completed` | `cancelled`
- `Booking.paymentStatus`: `unpaid` | `paid`
- `Coupon.discountType`: `free` | `percent`
- `PriceTier`: `free` | `discounted` | `standard` | `premium`
- `TimeSlotTier`: `premium` | `regular` | `discounted` | `unavailable`
- `TimeSlotBooking.status`: `pending` | `confirmed` | `completed` | `cancelled`
- `CallInvitation.status`: `sent` | `scheduled` | `completed` | `expired`
- `CallInvitation.callType`: `free` | `discounted`
- `User.role`: `admin` | `manager` | `advisor` | `student` | `client` | `candidate`
- `UserMessage.status`: `sent` | `read` | `replied`

---

## 4. P1 — iOS Quick Actions (high value, low complexity)

These are the controls the admin uses most often on the web. iOS should absolutely have a view and, where safe, an edit control for each.

### 4.1 Payments & Default Booking Price

**Collection:** `payment_config`  
**Doc ID:** `booking`  
**Field:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | always `booking` | `booking` |
| `defaultPrice` | number | fallback price for a consultation if no offering override | `4.5` (USD, major unit) |
| `updatedAt` | Date | last edit timestamp | `Timestamp` / ISO |

**Web control:** `TapConfigManagement` page.

**iOS suggestion:**

- **View:** Dashboard card showing current default price.
- **Control:** Number input with `step="0.01"`, `min="0"`, save button writes `payment_config/booking` with `{ defaultPrice, updatedAt: new Date() }`.

> Do not rely on editing this for service-specific prices; `service_offerings` can override `priceUSD`.

---

### 4.2 Tap Payment Config

**Collection:** `tap_config`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | auto-generated | `Date.now().toString()` |
| `secretKey` | string | Tap secret key | `sk_test_...` / `sk_live_...` |
| `publicKey` | string | Tap public key | `pk_test_...` / `pk_live_...` |
| `environment` | string | `test` or `production` | `test` / `production` |
| `isActive` | boolean | only one should be active per environment | `true` / `false` |
| `createdAt` | Date | creation timestamp | `Timestamp` / ISO |

**Web controls:**

- List configs with environment badge, active badge, masked secret.
- Toggle `isActive`.
- Add/edit/delete modal with `environment`, `publicKey`, `secretKey`, `isActive`.
- Test payment button calls `createTapTestPayment(amount, currency)`.

**iOS suggestion:**

- **View:** A list of Tap configs. Each cell shows `environment` (pill), `isActive` (pill), and public key prefix.
- **Control:** Edit toggle to activate/deactivate. iOS should **not** store raw keys in the app bundle; but it can expose an `Add Config` form that writes to Firestore.
- **Test payment:** `amount` number + `currency` picker (`USD`, `KWD`, `SAR`, `AED`, `JOD`) calls `createTapTestPayment` and opens the returned `url` in `SFSafariViewController`.

---

### 4.3 Bookings

**Collection:** `bookings`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | e.g. `1720000000000abc123` |
| `userId` | string? | linked account UID | systemUsers/{uid} `id` |
| `email` | string | client email | `mohammad@example.com` |
| `name` | string | client name | `Mohammad` |
| `company` | string? | optional company | `ABC` |
| `serviceOfferingId` | string | FK to `service_offerings` | offering doc id |
| `serviceType` | string | offering type | `mentorship_coffee`, `discovery_call_30`, etc. |
| `scheduledAt` | Date | ISO / Timestamp | `2026-07-20T10:00:00.000Z` |
| `durationMinutes` | number | length | `15`, `30`, `60` |
| `status` | string | booking status | `pending`, `confirmed`, `completed`, `cancelled` |
| `priceUSD` | number | amount paid | `4.5` |
| `paymentStatus` | string | `unpaid` / `paid` | `unpaid`, `paid` |
| `tapChargeId` | string? | Tap charge ID | `chg_...` |
| `tapUrl` | string? | payment URL | `https://checkout.tap.company/...` |
| `sessionLink` | string? | Jitsi/meeting link | `https://meet.jit.si/robotics-sciences-{id}` |
| `notes` | string? | admin/client notes | free text |
| `createdAt` | Date | booking created | `Timestamp` |

**Web controls:**

- `BookingsManagement` page: filter by status, date, search by name/email.
- Update `status` with dropdown.
- Delete booking.
- Send `sendBookingStatus` email on status change.
- New `BookingsPage` public page allows clients to search by email/name, continue payment, resend confirmation, copy session link.

**iOS suggestion:**

- **Dashboard:** "Today" card counting `confirmed` bookings and revenue.
- **View:** List with `status` and `paymentStatus` pills, date, time, service name.
- **Control:**
  - Tap a booking to see details.
  - Action sheet with status change: `pending → confirmed → completed` or `cancelled`.
  - If `paymentStatus == unpaid`, button **Continue payment** calls `resumeBookingPayment(bookingId)` and opens URL in `SFSafariViewController`.
  - Button **Resend confirmation** calls `resendBookingEmail(bookingId, email)`.
  - Button **Join session** opens `sessionLink` or `https://meet.jit.si/robotics-sciences-{id}`.
  - Button **Edit session link** updates `booking.sessionLink`.

**Queries:**

- Today's bookings: `bookings` where `scheduledAt` is today and `status != 'cancelled'`.
- Upcoming: `bookings` where `scheduledAt >= startOfDay` sorted by `scheduledAt`.
- Recent payments: `bookings` where `paymentStatus == 'paid'`.
- Search by `email` or `name` substring.

---

### 4.4 Consultant Availability

**Collection:** `availability`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `dayOfWeek` | number | 0–6 | `0` = Sunday |
| `startTime` | string | `HH:mm` | `"10:00"` |
| `endTime` | string | `HH:mm` | `"18:00"` |
| `bufferMinutes` | number | gap between slots | `15` |
| `maxBookingsPerDay` | number | max bookings per day | `5` |
| `priceTier` | string | `PriceTier` | `free`, `discounted`, `standard`, `premium` |

**Web controls:**

- `AvailabilityManagement` card grid with create/edit/delete modal.
- "Seed" button creates default Sun–Thu 10:00–18:00.

**iOS suggestion:**

- **View:** A 7-day list with one card per day, showing start/end, buffer, max bookings, price tier pill.
- **Control:** Edit/add modal with day picker (`UISegmentedControl` 0–6), start/end time pickers, buffer stepper, max bookings stepper, price tier segmented (`free | discounted | standard | premium`).
- **Recommended:** A "Calendar" tab with this as a filter layer.

**Tier pricing effect:**

- `free`: `0` regardless of base price.
- `discounted`: `0.5 × basePrice`.
- `standard`: `1 × basePrice`.
- `premium`: `1.25 × basePrice`.

---

### 4.5 Calendar Protection Blocks

**Collection:** `calendar_blocks`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `dayOfWeek` | number | 0–6 | `0` = Sunday |
| `startTime` | string | `HH:mm` | `"09:00"` |
| `endTime` | string | `HH:mm` | `"12:00"` |
| `label` | string | EN label | `"Business Development"` |
| `labelAr` | string? | AR label | `"تطوير الأعمال"` |
| `isProtected` | boolean | non-sellable | `true` |
| `createdAt` | Date | timestamp | `Timestamp` |

**Web controls:** `CalendarBlocksManagement` with add/edit/delete.

**iOS suggestion:**

- **View:** Same list as availability but with a red/gray pill `isProtected`.
- **Control:** Form with day, start/end time, labels (EN + AR), and a toggle "Non-sellable".

---

### 4.6 Coupon Codes

**Collection:** `coupons`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `code` | string | uppercase unique code | `ABC12345` |
| `discountType` | string | `free` or `percent` | `free` / `percent` |
| `discountValue` | number | 100 for free, 1–99 for percent | `100` or `50` |
| `maxUses` | number | max allowed uses | `1` |
| `usedCount` | number | counter incremented on use | `0` |
| `expiresAt` | Date? | optional expiry | `Timestamp` |
| `isActive` | boolean | toggle | `true` / `false` |
| `notes` | string? | internal notes | free text |
| `createdAt` | Date | timestamp | `Timestamp` |

**Web controls:** `CouponManagement` with create, toggle active, delete, copy code.

**iOS suggestion:**

- **View:** Grid/list of coupons with code, discount pill, usage `usedCount / maxUses`, expiry date.
- **Control:**
  - Create button: generate random 8-char code (A–Z, 2–9), discount type picker, percent slider (if percent), max uses, expiry date picker, notes.
  - Toggle `isActive`.
  - Tap to copy code to clipboard.

---

## 5. P2 — iOS Moderate Admin Controls

### 5.1 Service Offerings

**Collection:** `service_offerings`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `name` | string | EN title | `Discovery Call — 30 min` |
| `nameAr` | string? | AR title | `مكالمة اكتشاف — 30 دقيقة` |
| `description` | string | EN description | free text |
| `descriptionAr` | string? | AR description | free text |
| `type` | string | offering type | see enum below |
| `audience` | string | `student` / `graduate` / `company` / `robotics` / `learner` / `builder` / `talent` | |
| `category` | string | `career` / `training` / `advisory` / `technical` / `robotics` / `mentorship` / `builder` / `learner` / `company` | |
| `priceUSD` | number | price | `35` |
| `originalPriceUSD` | number? | strikethrough price | `0` |
| `durationMinutes` | number | length | `30` |
| `isActive` | boolean | show/hide | `true` |
| `outcome` | string? | what client gets | free text |
| `outcomeAr` | string? | AR outcome | free text |
| `deliverables` | string[]? | list of deliverables | `["30-min video call", "Scope discussion"]` |
| `deliverablesAr` | string[]? | AR list | `[]` |
| `duration` | string? | human duration | `"30 min"`, `"4 weeks"` |
| `commitment` | string? | frequency | `"1 session"`, `"4 sessions"` |
| `pricePer` | string? | `package` / `month` / `hour` / `session` / `project` | |
| `sessionCount` | number? | number of sessions in the package | `6` |
| `featured` | boolean? | show on discover | `true` / `false` |
| `order` | number? | sort order | `1` |
| `createdAt` | Date | timestamp | `Timestamp` |

**`type` enum values:**

```
coffee_time, company_assessment, membership, course_access, career_roadmap,
interview_success, portfolio_builder, team_assessment, custom_training,
advisory_retainer, ai_marketing_system, ai_team_empowerment, job_search_strategy,
cv_makeover, career_transition, discovery_call_30, discovery_call_60,
robotics_mvp, mentorship_1x6, mentorship_2x3, mentorship_4x, mentorship_coffee,
rescue_call, sprint_supervision, full_supervision, direction_call, steady_push,
intensive_care
```

**Web controls:** `ServiceOfferingsManagement` with full CRUD, AI Arabic translation, active toggle, order, featured.

**iOS suggestion:**

- **View:** List of offerings with active pill, price, duration, category pill.
- **Control:**
  - Create/edit form with all fields.
  - Toggle `isActive` from the list.
  - Reorder via drag handle.
  - Separate `Discover` section toggles for `featured`.

> iOS read `isActive` offerings for the booking flow and use `priceUSD` / `durationMinutes`.

---

### 5.2 Time Slot Tiers (Student Booking)

**Collection:** `time_slot_config`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `dayOfWeek` | number | 0–6 | `0` = Sunday |
| `startHour` | number | 0–23 | `10` |
| `endHour` | number | 0–23 | `16` |
| `tier` | string | `premium` / `regular` / `discounted` / `unavailable` | |
| `slotDurationMinutes` | number | `30`, `45`, `60`, `90`, `120` | `60` |
| `priceJOD` | number | JOD price | `20` |
| `maxSeats` | number | max bookings per slot | `3` |
| `label` | string? | optional label | `"Evening Rush"` |
| `labelAr` | string? | AR label | `"المساء"` |
| `createdAt` | Date | timestamp | `Timestamp` |

**Web controls:** `TimeSlotConfigManagement` with day grouping, tier pills, add/edit/delete.

**iOS suggestion:**

- **View:** Calendar-like grid grouped by day, each cell `10:00–16:00 · Regular · 3 seats · 20 JOD`.
- **Control:**
  - Modal with day picker, start/end hour pickers, 4-tier segmented control.
  - If tier `unavailable`, price/seats disabled / zero.
  - Duration picker (`30/45/60/90/120`), max seats `1–20`, price `JOD` step `0.5`.

---

### 5.3 Time Slot Bookings

**Collection:** `time_slot_bookings`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `studentId` | string | FK | systemUsers id |
| `studentName` | string | display name | `Ahmad` |
| `studentEmail` | string | email | `ahmad@example.com` |
| `slotDate` | string | `YYYY-MM-DD` | `2026-07-20` |
| `slotStartHour` | number | 0–23 | `10` |
| `slotEndHour` | number | 0–23 | `11` |
| `tier` | string | TimeSlotTier | `regular` |
| `originalPriceJOD` | number | original JOD | `20` |
| `finalPriceJOD` | number | paid JOD | `20` |
| `discountPercent` | number | 0–100 | `0` |
| `callInvitationId` | string? | FK to `call_invitations` | — |
| `status` | string | `pending` / `confirmed` / `completed` / `cancelled` | |
| `notes` | string? | — | free text |
| `createdAt` | Date | timestamp | `Timestamp` |

**iOS suggestion:**

- **View:** List of student bookings for selected date, with status pill.
- **Control:** Confirm / cancel / complete from action sheet. Add/edit `notes`.

---

### 5.4 Call Invitations

**Collection:** `call_invitations`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `submissionId` | string | FK to `job_submissions` | — |
| `studentName` | string | name | `Ahmad` |
| `studentEmail` | string | email | `ahmad@example.com` |
| `callType` | string | `free` / `discounted` | `free` |
| `discountPercent` | number | if discounted, 0–100 | `50` |
| `personalMessage` | string? | custom message | free text |
| `status` | string | `sent` / `scheduled` / `completed` / `expired` | |
| `bookedSlotId` | string? | FK to `time_slot_bookings` | — |
| `createdAt` | Date | timestamp | `Timestamp` |
| `expiresAt` | Date? | optional expiry | `Timestamp` |

**iOS suggestion:**

- **View:** List of invitations with status.
- **Control:** Create invitation from a CV submission: choose `callType`, `discountPercent`, `personalMessage`, `expiresAt`. Status updates when student books.

---

### 5.5 Users / System Users

**Collection:** `systemUsers`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | Firebase Auth UID | — |
| `name` | string | display name | `Mohammad` |
| `role` | string | `admin` / `manager` / `advisor` / `student` / `client` / `candidate` | |
| `email` | string? | email | `m@example.com` |
| `username` | string? | lowercase username | `mohammad` |
| `hasPassword` | boolean? | has password set | `true` / `false` |
| `projectIds` | string[] | project access | `[]` |
| `permissions` | object | granular booleans | see below |

**`permissions` object (booleans):**

```
canViewDashboard, canViewFinancials, canViewProjectManagement,
canViewCRM, canViewContracts, canViewTransactions, canViewReports,
canManageSubscriptions, canGenerateReports, canEdit, canDelete, canComment,
canManageUsers, canManageProjects
```

**Web controls:** `UserManagement` lists users, create, edit role, impersonate, delete.

**iOS suggestion:**

- **View:** User list with role pill and email.
- **Control:**
  - Create account (or use `setupAdmin` callable).
  - Edit role, permissions toggles, project assignment.
  - Impersonate is web-only for safety; do not expose on iOS unless needed.

---

### 5.6 Role Definitions

**Collection:** `site_config`  
**Doc ID:** `roles`

**Field:**

| Field | Type | Description |
|---|---|---|
| `id` | string | `roles` |
| `roles` | array | custom `RoleDefinition` objects |

Each role object:

| Field | Type | Description |
|---|---|---|
| `id` | string | `admin`, `advisor`, `client`, `student` or custom |
| `name` | string | display name |
| `description` | string | description |
| `permissions` | object | same `permissions` as `User` |
| `editable` | boolean | can this role be edited |

**Web controls:** `RoleManagement` creates custom roles with permission toggles.

**iOS suggestion:**

- **View:** List of roles.
- **Control:** Create custom role with name, description, permission toggles; save to `site_config/roles`.

---

### 5.7 CV Submissions / Talent Pool

**Collection:** `job_submissions`

**Schema (key fields):**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `fullName` | string | name | `Ahmad` |
| `email` | string | email | `ahmad@example.com` |
| `phone` | string? | phone | — |
| `role` | string | applied role | `ROS Intern` |
| `message` | string? | cover message | — |
| `cvFileName` | string | uploaded CV filename | `cv.pdf` |
| `cvFileType` | string | MIME type | `application/pdf` |
| `cvFileSize` | number | bytes | `123456` |
| `cvBase64` | string | base64 file | `data:application/pdf;base64,...` |
| `extractedText` | string? | OCR text | free text |
| `status` | string | see enum | `new`, `reviewing`, `shortlisted`, `invited`, `preparing`, `interested`, `declined`, `survey_completed`, `enrolled`, `archived` |
| `notes` | string? | admin notes | free text |
| `reviewedAt` | Date? | — | `Timestamp` |
| `reviewedBy` | string? | admin user id | — |
| `invitedAt` | Date? | — | `Timestamp` |
| `invitedBy` | string? | admin user id | — |
| `submittedAt` | Date | — | `Timestamp` |
| `offerSentAt` | Date? | — | `Timestamp` |
| `offerType` | string? | `coffee_time`, `membership`, `course_access`, `full_funnel` | |
| `isGraduate` | boolean? | — | `true` / `false` |
| `jobType` | string / string[] | `full-time`, `part-time`, `freelance`, `internship`, `course` | |
| `careerId` | string? | FK to `career_opportunities` | — |
| `acceptsUnpaidInternship` | boolean? | — | `true` / `false` |
| `extractedData` | object? | `CVExtractedData` | see below |
| `preparedCVIds` | string[]? | FKs to `prepared_cvs` | — |
| `cvAnalysis` | object? | AI analysis cache | `{ skills, strongTopics, gapTopics, suggestedMilestones, summary, analyzedAt, model }` |
| `deviceInfo` | object? | visitor metadata | `userAgent, platform, browser, os, deviceType, screenResolution, language, timezone, referrer, pageUrl, ipAddress` |

**Web controls:** `Submissions` / `TalentPoolManagement` / `CVPreparation` / `SubmissionResponse`.

**iOS suggestion:**

- **View:** Submission list with status pill and name. Filter by status.
- **Control:**
  - Tap to view CV metadata and `extractedData` (name, email, phone, skills, etc.).
  - Change status via dropdown.
  - Button **Extract CV Data** calls `extractCVData(submissionId)`.
  - Button **Send offer** with `offerType` picker and calls `sendOfferEmail`.
  - Button **Create call invitation** creates a `call_invitations` doc.

---

### 5.8 CRM Contacts (Client CRM)

**Collection:** `crm_contacts` (hardcoded string, not in `STORAGE_KEYS`)

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `submissionId` | string | FK to `job_submissions` | — |
| `name` | string | name | — |
| `email` | string | email | — |
| `phone` | string? | phone | — |
| `role` | string | role | — |
| `status` | string | `prospect` / `enrolled` / `active` / `alumni` / `inactive` | |
| `proposedCourses` | string[] | course ids | `[]` |
| `enrolledCourses` | string[] | course ids | `[]` |
| `proposedServices` | string[] | service offering ids | `[]` |
| `cvFileName` | string? | — | — |
| `cvBase64` | string? | — | — |
| `cvNotes` | string? | — | free text |
| `accountCreated` | boolean | — | `true` / `false` |
| `accountId` | string? | systemUsers uid | — |
| `emailSentAt` | Date? | — | `Timestamp` |
| `responseMessage` | string? | — | free text |
| `responseToken` | string? | — | — |
| `lastResponseType` | string? | — | — |
| `lastResponseAt` | Date? | — | `Timestamp` |
| `notes` | string? | — | free text |
| `createdAt` | Date | — | `Timestamp` |
| `updatedAt` | Date? | — | `Timestamp` |

**iOS suggestion:**

- **View:** CRM list with status, filter by status.
- **Control:** Edit status, add `notes`, propose courses/services (multi-select from `courses` and `service_offerings`).

---

### 5.9 Student CRM

**Collection:** `student_crm`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `submissionId` | string | FK to `job_submissions` | — |
| `name` | string | name | — |
| `email` | string | email | — |
| `phone` | string? | phone | — |
| `role` | string | role | — |
| `status` | string | `invited` / `active` / `enrolled` / `alumni` / `inactive` | |
| `cvFileName` | string? | — | — |
| `cvBase64` | string? | — | — |
| `cvNotes` | string? | — | free text |
| `accountCreated` | boolean | has account | `true` / `false` |
| `accountId` | string? | systemUsers uid | — |
| `callInvitationId` | string? | FK | — |
| `callType` | string? | `free` / `discounted` | |
| `callDiscountPercent` | number? | 0–100 | — |
| `callStatus` | string? | `sent` / `scheduled` / `completed` / `expired` | |
| `proposedCourses` | string[] | course ids | `[]` |
| `enrolledCourses` | string[] | course ids | `[]` |
| `emailSentAt` | Date? | — | `Timestamp` |
| `responseMessage` | string? | — | free text |
| `notes` | string? | — | free text |
| `mentorshipUnlocked` | boolean? | — | `true` / `false` |
| `mentorshipUnlockedAt` | Date? | — | `Timestamp` |
| `createdAt` | Date | — | `Timestamp` |
| `updatedAt` | Date? | — | `Timestamp` |

**iOS suggestion:**

- **View:** Student CRM list with status pill.
- **Control:** Change status, unlock mentorship toggle, add notes, propose/enroll courses.

---

### 5.10 Session Outcomes

**Collection:** `session_outcomes`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `bookingId` | string | FK to `bookings` | — |
| `clientName` | string | name | — |
| `clientEmail` | string | email | — |
| `serviceName` | string | service | — |
| `serviceType` | string | service type | — |
| `sessionDate` | Date | `Timestamp` | — |
| `outcomeTextEn` | string? | EN outcome | free text |
| `outcomeTextAr` | string? | AR outcome | free text |
| `clientResponse` | string? | client feedback | free text |
| `clientResponseAt` | Date? | — | `Timestamp` |
| `published` | boolean | show to client | `true` / `false` |
| `publishedAt` | Date? | — | `Timestamp` |
| `createdAt` | Date | — | `Timestamp` |

**iOS suggestion:**

- **View:** Outcomes per completed booking.
- **Control:** Add/edit outcome text (EN + AR), publish toggle, delete.

---

### 5.11 Price Escalation Rules

**Collection:** `price_escalation_rules`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `offeringType` | string | matches `ServiceOffering.type` | `mentorship_coffee` |
| `triggerFillCount` | number | number of fills before raising price | `5` |
| `priceIncreasePercent` | number | percent increase | `10` |
| `currentPriceUSD` | number | current escalated price | `15` |
| `lastAppliedAt` | Date? | — | `Timestamp` |
| `consecutiveFills` | number | counter | `0` |
| `enabled` | boolean | active | `true` / `false` |
| `createdAt` | Date | — | `Timestamp` |

**iOS suggestion:**

- **View:** List of rules with `offeringType`, `currentPriceUSD`, `consecutiveFills`, `triggerFillCount`.
- **Control:** Enable/disable toggle, edit trigger and increase percent, reset counter.

---

### 5.12 Contact Submissions

**Collection:** `contact_submissions`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `name` | string | — | — |
| `email` | string | — | — |
| `company` | string? | — | — |
| `message` | string | — | free text |
| `type` | string | `general` / `company` / `service` / `talent` | |
| `status` | string | `new` / `replied` / `archived` | |
| `notes` | string? | admin notes | free text |
| `repliedAt` | Date? | — | `Timestamp` |
| `submittedAt` | Date | — | `Timestamp` |

**iOS suggestion:**

- **View:** Inbox list with status and type.
- **Control:** Mark `replied` or `archived`, add `notes`, call `sendContactReply(contactId, replyMessage)` to reply.

---

### 5.13 User-to-Admin Messages

**Collection:** `user_messages`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `userId` | string | systemUsers uid | — |
| `userEmail` | string | — | — |
| `userName` | string | — | — |
| `subject` | string | — | — |
| `body` | string | message | free text |
| `adminReply` | string? | admin reply | free text |
| `repliedAt` | Date? | — | `Timestamp` |
| `repliedBy` | string? | admin uid | — |
| `status` | string | `sent` / `read` / `replied` | |
| `createdAt` | Date | — | `Timestamp` |

**iOS suggestion:**

- **View:** Message inbox.
- **Control:** Reply in-line, set status to `replied`.

---

### 5.14 Portal Messages

**Collection:** `portal_messages`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `senderId` | string | admin uid | — |
| `senderName` | string | admin name | — |
| `title` | string | message title | — |
| `body` | string | EN body | — |
| `bodyAr` | string? | AR body | — |
| `audience` | string | `all` / `students` / `clients` / `candidates` / `specific` | |
| `targetEmails` | string[]? | used when `audience == 'specific'` | `[]` |
| `priority` | string | `normal` / `important` / `urgent` | |
| `isRead` | boolean | at least one read | `true` / `false` |
| `readBy` | string[] | emails of readers | `[]` |
| `readAt` | Date? | — | `Timestamp` |
| `expiresAt` | Date? | — | `Timestamp` |
| `createdAt` | Date | — | `Timestamp` |

**iOS suggestion:**

- **View:** List of announcements by priority.
- **Control:** Create announcement with title, body (EN + AR), audience picker, priority, expiry date.

---

### 5.15 Careers / Job Postings

**Collection:** `career_opportunities`

**Schema:**

| Field | Type | Description | Expected value |
|---|---|---|---|
| `id` | string | generated | — |
| `title` | string | EN title | `Robotics Intern` |
| `titleAr` | string? | AR title | — |
| `department` | string | department | `Engineering` |
| `location` | string | location | `Amman / Remote` |
| `type` | string | `full-time` / `part-time` / `freelance` / `internship` / `course` | |
| `description` | string | EN | free text |
| `descriptionAr` | string? | AR | free text |
| `isActive` | boolean | show on site | `true` / `false` |
| `createdAt` | Date | — | `Timestamp` |
| `updatedAt` | Date? | — | `Timestamp` |
| `aiFormatted` | boolean? | AI formatted | `true` / `false` |

**iOS suggestion:**

- **View:** Careers list with active pill.
- **Control:** Create/edit, toggle active, delete.

---

## 6. P3 — Nice-to-Have iOS Controls

### 6.1 Courses

**Collection:** `courses`

**Key fields:**

| Field | Type | Expected value |
|---|---|---|
| `id` | string | generated |
| `title` | string | EN title |
| `titleAr` | string? | AR title |
| `description` | string | EN |
| `descriptionAr` | string? | AR |
| `instructor` | string | name |
| `instructorAr` | string? | AR |
| `category` | string | free text |
| `level` | string | `beginner` / `intermediate` / `advanced` |
| `price` | number | USD |
| `isFree` | boolean | true / false |
| `packageDiscountPercent` | number | 0–100 |
| `isPublished` | boolean | true / false |
| `thumbnailUrl` | string? | image URL |
| `totalDurationMinutes` | number | computed |
| `lectureCount` | number | computed |
| `rating` | number | 0–5 |
| `studentCount` | number | count |
| `hasPlacementTest` | boolean | true / false |
| `tags` | string[] | `[]` |
| `createdAt` | Date | Timestamp |
| `updatedAt` | Date? | Timestamp |

**iOS suggestion:**

- **View:** Course list with publish toggle.
- **Control:** Create/edit basic fields, publish/unpublish. Lectures are stored in `lectures` collection (by `courseId`); iOS can manage them later if needed.

---

### 6.2 Membership Plans

**Collection:** `membership_plans`

**Key fields:**

| Field | Type | Expected value |
|---|---|---|
| `id` | string | generated |
| `name` | string | EN |
| `nameAr` | string? | AR |
| `description` | string | EN |
| `descriptionAr` | string? | AR |
| `price` | number | USD |
| `billingCycle` | string | `monthly` / `yearly` / `one-time` |
| `features` | string[] | `[]` |
| `featuresAr` | string[]? | `[]` |
| `isPopular` | boolean | true / false |
| `isActive` | boolean | true / false |
| `courseIds` | string[] | course ids |
| `createdAt` | Date | Timestamp |

**iOS suggestion:**

- **View:** Plan list with `isPopular` and `isActive` pills.
- **Control:** Create/edit, toggle active/popular.

---

### 6.3 Videos

**Collection:** `video_tutorials`

**Key fields:**

| Field | Type | Expected value |
|---|---|---|
| `id` | string | generated |
| `title` | string | EN |
| `titleAr` | string? | AR |
| `description` | string | EN |
| `descriptionAr` | string? | AR |
| `videoUrl` | string | URL |
| `thumbnailUrl` | string? | URL |
| `language` | string | `en` / `ar` / `both` |
| `category` | string | `latex-compiler` / `cv-building` / `website-tour` / `courses` / `general` |
| `durationMinutes` | number | — |
| `order` | number | sort order |
| `isPublished` | boolean | true / false |
| `uploadedBy` | string | admin name |
| `createdAt` | Date | Timestamp |

**iOS suggestion:**

- **View:** Video list with published pill.
- **Control:** Toggle publish, view metadata. Upload from iOS is not recommended unless the backend supports direct storage.

---

### 6.4 Skill Question Bank

**Collection:** `skill_questions`

**Key fields:**

| Field | Type | Expected value |
|---|---|---|
| `id` | string | generated |
| `topic` | string | `Python`, `ROS`, `Computer Vision`, `Control Systems`, `Embedded Systems` |
| `topicAr` | string? | AR |
| `question` | string | EN question |
| `questionAr` | string? | AR |
| `options` | string[] | 4 options |
| `optionsAr` | string[]? | AR options |
| `correctIndex` | number | 0–3 |
| `difficulty` | string | `basic` / `intermediate` / `advanced` |
| `isActive` | boolean | true / false |
| `timesAsked` | number? | counter |
| `timesCorrect` | number? | counter |
| `createdAt` | Date | Timestamp |
| `createdBy` | string? | admin uid |

**iOS suggestion:**

- **View:** Question list by topic.
- **Control:** Add/edit question, options, correct answer, difficulty, active toggle.

---

### 6.5 Discover Quiz Results

**Collection:** `discover_quiz_results`

**Key fields:**

| Field | Type | Description |
|---|---|---|
| `id` | string | generated |
| `email` | string? | taker email |
| `name` | string? | taker name |
| `submissionId` | string? | FK |
| `answers` | array | `{ questionId, topic, selectedIndex, correct, selfKnown }` |
| `topicScores` | object | `{ topic: { correct, total } }` |
| `strongTopics` | string[] | — |
| `improveTopics` | string[] | — |
| `recommendedTopics` | string[] | — |
| `takenAt` | Date | Timestamp |

**iOS suggestion:**

- **View:** Read-only list of quiz results with topic scores.

---

### 6.6 Site / Website Config

**Collection:** `site_config`

**Doc IDs:**

- `website_sections` — controls public navigation visibility.
- `roles` — role definitions (see 5.6).

**`website_sections` fields:**

| Field | Type | Description |
|---|---|---|
| `id` | string | `website_sections` |
| `showNews` | boolean | show news section |
| `showProjects` | boolean | show projects section |
| `showCourses` | boolean | show courses section |
| `showMembership` | boolean | show membership section |
| `showTutorials` | boolean | show tutorials section |

**iOS suggestion:**

- **View:** Settings toggles for each section.
- **Control:** Toggle any boolean and write to `site_config/website_sections`.

---

### 6.7 Analytics

**Collections:** `analytics_events`, `analytics_sessions`

**Event schema:**

| Field | Type | Description |
|---|---|---|
| `id` | string | generated |
| `sessionId` | string | FK |
| `type` | string | `page_view`, `click`, `form_submit`, `scroll_depth`, `session_start`, `session_end` |
| `page` | string | path |
| `elementId` | string? | — |
| `elementText` | string? | — |
| `elementPosition` | object? | `{ x, y }` |
| `scrollDepth` | number? | — |
| `duration` | number? | ms |
| `referrer` | string? | — |
| `timestamp` | Date | Timestamp |

**Session schema:**

| Field | Type | Description |
|---|---|---|
| `id` | string | generated |
| `sessionId` | string | — |
| `language` | string | `en` / `ar` |
| `browser` | string | — |
| `os` | string | — |
| `device` | string | `desktop` / `mobile` / `tablet` |
| `screenResolution` | string | — |
| `viewport` | string | — |
| `timezone` | string | — |
| `country` | string? | — |
| `startedAt` | Date | Timestamp |
| `lastActivityAt` | Date | Timestamp |

**iOS suggestion:**

- **View:** Simple dashboard card with total page views, sessions, and device breakdown. Read-only.

---

### 6.8 Latex Templates

**Collection:** `latex_templates`

**Schema:**

| Field | Type | Description |
|---|---|---|
| `id` | string | generated with `custom-` prefix |
| `name` | string | template name |
| `description` | string | description |
| `source` | string | LaTeX source |
| `category` | string | category |
| `isBuiltIn` | boolean | `true` / `false` |
| `createdAt` | number | unix ms |
| `createdBy` | string? | admin uid |

**iOS suggestion:**

- **View:** List templates.
- **Control:** Read-only on small screen; editing LaTeX on iOS is possible with a code editor but not high priority.

---

### 6.9 Agent Log

**Collection:** `agent_log`

**Doc ID:** `main`

**Fields:**

| Field | Type | Description |
|---|---|---|
| `id` | string | `main` |
| `content` | string | markdown content |
| `updatedAt` | string | ISO timestamp |

**iOS suggestion:**

- **View:** Read-only note.

---

## 7. Master collection reference

| Collection ID | Web page | iOS priority | What iOS should build |
|---|---|---|---|
| `payment_config` | `TapConfigManagement` | P1 | Default price card + edit |
| `tap_config` | `TapConfigManagement` | P1 | Config list, toggle active, test payment |
| `bookings` | `BookingsManagement`, `BookingsPage` | P1 | Dashboard today, list, status change, continue payment, resend email, join session |
| `availability` | `AvailabilityManagement` | P1 | Weekly availability list, add/edit/delete |
| `calendar_blocks` | `CalendarBlocksManagement` | P1 | Protected blocks list, add/edit/delete |
| `coupons` | `CouponManagement` | P1 | Coupon list, create, toggle, copy code |
| `service_offerings` | `ServiceOfferingsManagement` | P2 | List, CRUD, toggle active, sort order |
| `time_slot_config` | `TimeSlotConfigManagement` | P2 | Tier-based slot list, add/edit/delete |
| `time_slot_bookings` | `TimeSlotConfigManagement` | P2 | Student bookings list, confirm/cancel/complete |
| `call_invitations` | `BookingInvitations` | P2 | Create call invite, view status |
| `systemUsers` | `UserManagement` | P2 | User list, create, edit role/permissions |
| `site_config` / `roles` | `RoleManagement` | P2 | Role list, create custom role with permissions |
| `job_submissions` | `Submissions`, `TalentPoolManagement`, `CVPreparation` | P2 | CV list, status change, extract CV, send offer |
| `crm_contacts` | `CRMContacts` | P2 | Client CRM list, status/notes |
| `student_crm` | `StudentCRM` | P2 | Student CRM list, status/notes/courses |
| `session_outcomes` | `OutcomeManagement` | P2 | Add/edit outcomes per completed session |
| `price_escalation_rules` | `PriceEscalationManagement` | P2 | View/edit rules, enable/disable |
| `contact_submissions` | `ContactMessages` | P2 | Inbox, mark replied/archived, reply |
| `user_messages` | `UserMessages` / `MessagingCenter` | P2 | Admin reply to user messages |
| `portal_messages` | `MessagingCenter` / `PortalMessages` | P2 | Announcements create/list |
| `career_opportunities` | `CareersManagement` | P2 | Job posting list, CRUD, toggle active |
| `courses` | `CourseManagement` | P3 | Course list, CRUD, publish |
| `membership_plans` | `MembershipManagement` | P3 | Plan list, CRUD, toggle active |
| `video_tutorials` | `VideoManagement` | P3 | Video list, toggle publish |
| `skill_questions` | `QuestionBank` | P3 | Question bank list, CRUD |
| `discover_quiz_results` | `DiscoverQuiz` | P3 | Read-only results |
| `site_config` / `website_sections` | `WebsiteManager`, `Settings` | P3 | Toggle section visibility |
| `analytics_events` | `AdminAnalytics` | P3 | Read-only stats dashboard |
| `analytics_sessions` | `AdminAnalytics` | P3 | Read-only visitor sessions |
| `latex_templates` | `LatexCompiler` | P3 | Read-only list |
| `agent_log` | `AgentLog` | P3 | Read-only note |

---

## 8. Cloud Functions you can call from iOS

Use `Functions.functions().httpsCallable("functionName")` with `region: "us-central1"`.

| Function | Params | Returns | Use case |
|---|---|---|---|
| `createTapCheckout` | `{ serviceOfferingId, bookingDetails: { email, name, company?, scheduledAt, notes?, finalPrice?, couponCode? }, offeringData: { name, type, priceUSD, durationMinutes, isActive } }` | `{ success, chargeId, url }` | Client booking payment |
| `resumeBookingPayment` | `{ bookingId }` | `{ success, chargeId, url }` | Continue an unpaid booking |
| `resendBookingEmail` | `{ bookingId, email }` | `{ success, message }` | Resend confirmation/status email |
| `createTapTestPayment` | `{ amount, currency }` | `{ success, chargeId, url }` | Admin test payment |
| `sendBookingConfirmation` | `{ bookingId, clientName, clientEmail, serviceName, scheduledAt, durationMinutes, language }` | `{ success, message }` | Send confirmation email |
| `sendBookingStatus` | `{ email, name, status, scheduledAt, serviceName }` | `{ success }` | Notify booking status change |
| `sendBookingReminder` | `{}` | `{ success }` | Remind all upcoming bookings |
| `extractCVData` | `{ jobSubmissionId }` | `{ success, data }` | Parse CV in cloud |
| `sendOfferEmail` | `{ jobSubmissionId, offerType }` | `{ success, message }` | Offer coffee/membership/course |
| `sendCVFeedback` | `{ email, studentName, feedback, invitationId }` | `{ success }` | CV feedback email |
| `sendInvitationEmail` | `{ email, studentName, invitationId }` | `{ success }` | Invite email |
| `sendSubmissionResponse` | `{ email, subject, html }` | `{ success }` | Generic HTML response |
| `sendContactReply` | `{ contactId, replyMessage }` | `{ success }` | Reply to contact message |
| `sendPasswordSetupEmail` | `{ email, name, username }` | `{ success }` | Account ready email |
| `setupAdmin` | `{ email, name, username, password, setupKey }` | `{ success, uid }` | Create admin account |
| `syncBookingToCalendar` | `{ title, description, startISO, endISO, attendeeEmail, attendeeName }` | `{ success, eventId }` | Google Calendar sync |

---

## 9. Recommended iOS tab structure

| Tab | Screens | Collections / Functions |
|---|---|---|
| **Dashboard** | Today's bookings, revenue, counts, quick action buttons | `bookings`, `payment_config`, `analytics_events` |
| **Calendar** | Consultant availability, calendar blocks, time slot tiers, bookings overlay | `availability`, `calendar_blocks`, `time_slot_config`, `bookings` |
| **Bookings** | All bookings, search, status update, continue payment, resend, join session | `bookings`, `resumeBookingPayment`, `resendBookingEmail` |
| **Payments** | Tap config, default price, test payment | `tap_config`, `payment_config`, `createTapTestPayment` |
| **Coupons** | Coupon list and create | `coupons` |
| **People** | Users, CRM, student CRM, submissions, call invitations | `systemUsers`, `crm_contacts`, `student_crm`, `job_submissions`, `call_invitations` |
| **Services** | Service offerings, price escalation rules | `service_offerings`, `price_escalation_rules` |
| **Courses** | Courses, memberships, videos, questions | `courses`, `membership_plans`, `video_tutorials`, `skill_questions` |
| **Website** | Site section toggles, contact messages, careers, portal messages | `site_config`, `contact_submissions`, `career_opportunities`, `portal_messages` |
| **Settings** | Role definitions, user messages, analytics | `site_config/roles`, `user_messages`, `analytics_events` |

---

## 10. Security rules reminder

iOS must respect the same rules as the web:

- `systemUsers` is read by the authenticated user for their own profile.
- Admin users (role `admin` / `manager`) should be able to read/write all collections.
- Cloud Functions are the preferred way for payment/email actions because they need secrets.
- Do not call `sendBookingConfirmation` or `sendBookingStatus` without checking `permissions.canEdit`.

---

## 11. Next steps for iOS team

1. Start with the **Dashboard** and **Bookings** screens. They use the same `bookings` collection and `resumeBookingPayment` / `resendBookingEmail` functions.
2. Add **Calendar** (availability + blocks + time slot tiers) next.
3. Add **Payments** and **Coupons**.
4. Add **People** (users, CRM, submissions) for full admin workflow.
5. Use **SwiftUI** or **UIKit** with Firestore snapshots so data is live across web and iOS.

---

*This document is the handoff from web to iOS. If a field is missing or a behavior is unclear, search `src/types/index.ts` for the TypeScript interface and `src/utils/database.ts` for the DB methods used by the web portal.*
