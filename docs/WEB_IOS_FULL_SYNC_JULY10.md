# Web → iOS: Comprehensive Sync Update — July 10, 2026

**From:** Web team (Mohammad Shannak)
**To:** iOS team
**Date:** July 10, 2026
**Priority:** High — please read and respond

---

## Status

We still haven't received your `IOS_SYNC_RESPONSE_JULY9.md` (requested 48 hours ago). This document covers **everything** we've built since then. Please respond to both this and the original sync request.

---

## What We Built Since July 9

### 1. Invite-to-Call Flow (New)

When an admin reviews a submission, they can now choose "Invite to Call" as a response type. This:

- Creates a **`call_invitation`** record (free or discounted call)
- Creates a **`student_crm`** record (separate from client CRM)
- Sends a bilingual email postcard with call invitation details
- Student sees a banner in their portal Schedule tab with auto-applied discount

### 2. Seat-Booking Schedule System (New)

Students can now book time slots from a visual seat-booking UI in their portal:

- **Tier-based slots**: Premium, Regular, Discounted, Unavailable
- **Seat availability**: Shows "X/Y seats" per slot
- **Pricing**: Original price with strikethrough + discounted price
- **Week navigation**: Browse upcoming weeks
- **Auto-discount**: If student has a call invitation, discount applies automatically

### 3. Admin Time Slot Configuration (New)

Admin page to configure tier-based time slots per day of week:

- Day of week, start/end hour, tier, slot duration (30/45/60/90/120 min)
- Max seats per slot, price in JOD
- Optional EN/AR labels

### 4. Site-Wide Messaging System (New)

Admin can send announcements and targeted messages to portal users:

- **Audience targeting**: Everyone, Students, Clients, Candidates, or Specific Users (by email)
- **Priority levels**: Normal, Important, Urgent
- **Bilingual**: English + Arabic body
- **Expiry settings**: Days or never
- **Read tracking**: See who read each message
- Students see messages in their **Inbox** tab in the portal

### 5. Discover Page Improvements

- Header changed from "Work with Mohammad" to "Let's Build Together" / "لنبني معًا"
- Added Mohammad's photo as a circular avatar in the header
- Shortened descriptions to reduce bloating
- Added `leading-relaxed` and `dir="rtl"` for better Arabic spacing and alignment
- Simplified company form and quote form language

---

## Complete Collections List (Updated)

### Previously Shared Collections (unchanged)

| Collection | Purpose |
|---|---|
| `projects` | Project definitions (3 canonical: ADY, Riad Shannak, Website Tools) |
| `project_members` | User→Project membership |
| `systemUsers` | User accounts |
| `ady_stores`, `ady_transactions`, `ady_periods`, `ady_installments` | ADY accounting |
| `ady_crm` | ADY CRM leads |
| `comments` | Transaction/period comments |
| `timeline_entries` | Project timeline |
| `buildings` | Real estate buildings |
| `courses`, `lectures`, `enrollments` | Education |
| `placement_tests`, `placement_test_results` | Assessments |
| `video_tutorials`, `video_assignments` | Video content |
| `student_invitations` | Student invitations |
| `membership_plans`, `student_memberships` | Memberships |
| `cv_templates`, `prepared_cvs` | CV preparation |
| `quizzes`, `career_opportunities` | Assessments and job board |
| `service_offerings` | Service catalog |
| `bookings` | Scheduled sessions |
| `availability` | Consultant time slots |
| `tap_config` | Payment gateway config |
| `job_submissions` | Job applications |
| `contact_submissions` | Contact form entries |
| `email_responses` | Email response tracking |
| `email_inbox` | Email receipt processing |
| `widget_configs` | Dashboard widgets |
| `audit_log` | Action logging |
| `documents` | Portal documents |

### New Collections (5 — all added July 10)

#### `student_crm` — Student CRM (separate from client CRM)

```jsonc
{
  "id": "string",
  "submissionId": "string",
  "name": "string",
  "email": "string",
  "phone": "string?",
  "role": "string",
  "status": "invited | active | enrolled | alumni | inactive",
  "cvFileName": "string?",
  "cvBase64": "string?",
  "cvNotes": "string?",
  "accountCreated": "boolean",
  "accountId": "string?",
  "callInvitationId": "string?",
  "callType": "free | discounted?",
  "callDiscountPercent": "number?",
  "callStatus": "sent | scheduled | completed | expired?",
  "proposedCourses": ["string"],
  "enrolledCourses": ["string"],
  "emailSentAt": "Timestamp?",
  "responseMessage": "string?",
  "notes": "string?",
  "createdAt": "Timestamp",
  "updatedAt": "Timestamp?"
}
```

#### `call_invitations` — Call Invitation Records

```jsonc
{
  "id": "string",
  "submissionId": "string",
  "studentName": "string",
  "studentEmail": "string",
  "callType": "free | discounted",
  "discountPercent": "number",         // 0 if free, 10-100 if discounted
  "personalMessage": "string?",
  "status": "sent | scheduled | completed | expired",
  "bookedSlotId": "string?",
  "createdAt": "Timestamp",
  "expiresAt": "Timestamp?"            // 30 days from creation
}
```

#### `time_slot_config` — Tier-Based Time Slot Configuration

```jsonc
{
  "id": "string",
  "dayOfWeek": "number",               // 0 = Sunday, 6 = Saturday
  "startHour": "number",               // 0-23
  "endHour": "number",                 // 0-23
  "tier": "premium | regular | discounted | unavailable",
  "slotDurationMinutes": "number",     // 30, 45, 60, 90, 120
  "priceJOD": "number",
  "maxSeats": "number",
  "label": "string?",
  "labelAr": "string?",
  "createdAt": "Timestamp"
}
```

#### `time_slot_bookings` — Student Slot Bookings

```jsonc
{
  "id": "string",
  "studentId": "string",
  "studentName": "string",
  "studentEmail": "string",
  "slotDate": "string",                // "2026-07-15" (ISO date)
  "slotStartHour": "number",
  "slotEndHour": "number",
  "tier": "premium | regular | discounted",
  "originalPriceJOD": "number",
  "finalPriceJOD": "number",
  "discountPercent": "number",
  "callInvitationId": "string?",
  "status": "pending | confirmed | completed | cancelled",
  "notes": "string?",
  "createdAt": "Timestamp"
}
```

#### `portal_messages` — Site-Wide + Targeted Messages

```jsonc
{
  "id": "string",
  "senderId": "string",
  "senderName": "string",
  "title": "string",
  "body": "string",
  "bodyAr": "string?",
  "audience": "all | students | clients | candidates | specific",
  "targetEmails": ["string"]?,         // only when audience = "specific"
  "priority": "normal | important | urgent",
  "isRead": "boolean",
  "readBy": ["string"],                // emails of users who read it
  "readAt": "Timestamp?",
  "createdAt": "Timestamp",
  "expiresAt": "Timestamp?"
}
```

---

## Updated Tab Structure (28 tabs — was 26)

### Overview (6 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `dashboard` | Dashboard | accounting, real_estate, internal |
| `documents` | Documents | all |
| `student-portal` | Student Portal | website, education |
| `investor-portal` | Investor Portal | accounting, real_estate |
| `timeline` | Timeline | accounting |
| `riad-portal` | Riad Portal | real_estate |

### Finance (4 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `finance` | Finance | internal |
| `transactions` | Transactions | accounting |
| `budget` | Budget | accounting |
| `reports` | Reports | accounting, real_estate |

### Student (7 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `submissions` | Submissions | website |
| `student-overview` | Student Management | website, education |
| `course-management` | Courses | website, education |
| `membership-management` | Memberships | website, education |
| `cv-preparation` | CV Center | website, education |
| `video-hub` | Video Hub | website, education |
| `latex-compiler` | LaTeX Compiler | website, education |

### Operations (10 tabs — was 8)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `crm-contacts` | CRM Contacts | website, education |
| `email-inbox` | Email Inbox | website, education |
| `buildings` | Buildings | real_estate |
| `service-offerings` | Service Offerings | website, consulting |
| `proposal-generator` | Proposal Generator | website, consulting |
| `availability` | Availability | website, consulting |
| **`time-slot-config`** | Slot Tiers | website | ← NEW
| **`messaging`** | Messaging | website | ← NEW
| `bookings` | Bookings | website, consulting |
| `tap-config` | Tap Config | website |

### Website (1 tab)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `website-manager` | Website Manager | website |

### Administration (6 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `projects` | Projects | all |
| `user-management` | User Management | all |
| `role-management` | Role Management | all |
| `admin-settings` | Admin Settings | internal |
| `agent-log` | Agent Log | internal |
| `testing-lab` | Testing Lab | all |

### Student Portal Tabs (6 tabs — student-facing)
| Tab | Label | Purpose |
|-----|-------|---------|
| Overview | Overview | Student dashboard |
| **Inbox** | Inbox | Site-wide messages | ← NEW
| My CV | CV | CV preparation |
| Courses | Courses | Course catalog + enrollment |
| **Schedule** | Schedule | Seat-booking time slots | ← NEW
| Tools | Tools | LaTeX, video access (coming soon) |

---

## Cloud Functions (16 deployed — all `us-central1`)

| Function | Type | Purpose |
|----------|------|---------|
| `sendCVFeedback` | onCall | Email CV feedback to students |
| `sendInvitationEmail` | onCall | Email invitations to students |
| `sendSubmissionResponse` | onCall | Email response to applicants (now supports `invite_call` type) |
| `captureEmailResponse` | onRequest | Public endpoint for email button responses |
| `sendOfferEmail` | onCall | Email offer (coffee/membership/course) |
| `sendContactReply` | onCall | Reply to contact form submissions |
| `setupAdmin` | onCall | Create/repair admin accounts |
| `cleanupSystemUsers` | onCall | Remove duplicate user docs |
| `extractCVData` | onCall | Gemini AI CV extraction |
| `createTapCheckout` | onCall | Create Tap payment session |
| `tapWebhook` | onRequest | Handle Tap payment webhook |
| `sendBookingStatus` | onCall | Email booking status updates |
| `sendQuotation` | onCall | Send quotation to client |
| `captureQuotationResponse` | onRequest | Capture quotation response |
| `sendBookingConfirmation` | onCall | Email booking confirmation |
| `sendBookingReminder` | onCall | Email booking reminder |
| `processInboundEmail` | onRequest | Process inbound emails (Resend webhook) |

---

## Updated Types (for iOS reference)

### `SubmissionResponseType`
```typescript
'propose_course' | 'propose_service' | 'general_welcome' | 'invite_call' | 'rejection'
```

### New types
```typescript
type TimeSlotTier = 'premium' | 'regular' | 'discounted' | 'unavailable';
type MessageAudience = 'all' | 'students' | 'clients' | 'candidates' | 'specific';
type MessagePriority = 'normal' | 'important' | 'urgent';
```

### Course/Lecture pricing fields (added previously)
- `Course.packageDiscountPercent: number`
- `Lecture.price: number`
- `Lecture.isFree: boolean`

---

## Open Items — Still Need Your Response

From `WEB_IOS_SYNC_REQUEST_JULY9.md`:

1. **Share your complete Firebase collections list** organized by tab
2. **Share any features you built** that we don't know about
3. **Confirm 3-project structure**: ADY, Riad Shannak, Website Tools
4. **Align Discover page** — confirm collection schema and URL param support
5. **Check for duplicate collections** — flag any naming conflicts

From `IOS_FULL_SYNC_SPEC.md`:

6. **Roles**: `admin|manager|investor` vs our `admin|manager|advisor|client|student|candidate` — map `investor`→`client`?
7. **`service_offerings` → `consultations`**: Should we migrate? Or keep both?
8. **`ady_crm` vs `crm`**: Same or different? Please clarify
9. **Stripe vs Tap**: You mentioned Stripe for student payments (EUR). We use Tap for JOD. Do we need both?

---

## What's Next (Web Side)

- Placement test flow and lecture selection enrollment
- LaTeX compiler, public video access, and course catalog in student tools
- Student portal inbox notifications (badge count)
- Potential Stripe integration for EUR payments (if iOS confirms)

---

## Centralized Firestore Config (Implemented — July 10)

All shared configuration is now in Firestore. Both web and iOS read/write the same docs. When either platform changes something, the other sees it immediately.

### New collection: `site_config`

Single source of truth for cross-platform UI configuration. Each document has a unique ID:

#### Document: `site_config/discover_page` — Photo Position

```jsonc
{
  "id": "discover_page",
  "photoUrl": "/Mohammad-Shannak-Photo.jpeg",
  "photoScale": 1.2,            // zoom level (1.0 = 100%)
  "photoOffsetX": -10,          // pan X in px
  "photoOffsetY": 5,            // pan Y in px
  "photoUpdatedAt": "Timestamp",
  "updatedBy": "string"
}
```

**iOS**: Read on Discover page load. Apply `CGAffineTransform(scaleX:y:)` + translation to `UIImageView`. Admin can long-press to adjust. Write back to Firestore.

#### Document: `site_config/website_sections` — Section Visibility

```jsonc
{
  "id": "website_sections",
  "showNews": false,
  "showProjects": false,
  "showCourses": false,
  "showMembership": false,
  "showTutorials": false,
  "photoUpdatedAt": "Timestamp"
}
```

**iOS**: Read on app launch. Show/hide corresponding sections in the home screen. Admin can toggle from a settings panel. Write back to Firestore.

#### Document: `site_config/google_calendar` — Calendar Connection

```jsonc
{
  "id": "google_calendar",
  "calendarConnected": true,
  "calendarConnectedBy": "string",    // user ID
  "calendarConnectedAt": "Timestamp"
}
```

**iOS**: Read to show "Calendar connected" badge. When iOS connects via OAuth or EventKit, write `calendarConnected: true` here. Web sees it and knows sync is available.

**Note**: The actual OAuth token stays in iOS Keychain (not Firestore). Web keeps its own token in localStorage. The `calendarConnected` flag is just for UI status sharing.

#### Document: `site_config/roles` — Role Definitions

```jsonc
{
  "id": "roles",
  "roles": [
    {
      "id": "admin",
      "name": "Administrator",
      "description": "Full access",
      "permissions": { ... },
      "editable": false
    }
  ],
  "rolesUpdatedAt": "Timestamp"
}
```

**iOS**: Read to populate role picker in user management. When iOS admin edits roles, write back to Firestore. Web sees the same roles.

### New collection: `latex_templates` — Custom LaTeX/CV Templates

```jsonc
{
  "id": "custom-1234567890-abc123",
  "name": "My CV Template",
  "description": "Custom CV for robotics students",
  "source": "\\documentclass{article}...",
  "category": "custom",         // report | invoice | letter | article | custom
  "isBuiltIn": false,
  "createdAt": 1234567890,
  "createdBy": "string"
}
```

**iOS**: Read all docs to populate template picker in CV builder. When iOS user creates/edits/deletes a template, write to this collection. Web sees the same templates.

### What was migrated (web side)

| Config | Before | After |
|--------|--------|-------|
| Photo position | `localStorage` only | `site_config/discover_page` + localStorage fallback |
| Website sections | `localStorage` only | `site_config/website_sections` + localStorage fallback |
| Calendar status | `localStorage` only | `site_config/google_calendar` + localStorage fallback |
| Role definitions | `localStorage` only | `site_config/roles` + localStorage fallback |
| LaTeX templates | `localStorage` only | `latex_templates` collection + localStorage fallback |

### What stays local (not shared)

| Config | Why |
|--------|-----|
| `gemini_api_key` | Security — API key should stay on device or server-side only |
| `mentorship_unlocked` | Per-student state — should be on `student_crm` doc in Firestore |
| `portal_impersonating` | Web-only admin feature |
| Firebase offline cache | Internal to each platform's Firebase SDK |

### Pattern for iOS

All `site_config` reads follow the same pattern:

```swift
// Read
let doc = try await db.collection("site_config").document("discover_page").getDocument()
let scale = doc.data()?["photoScale"] as? Double ?? 1.0
let offsetX = doc.data()?["photoOffsetX"] as? Double ?? 0.0
let offsetY = doc.data()?["photoOffsetY"] as? Double ?? 0.0

// Write
try await db.collection("site_config").document("discover_page").setData([
    "photoScale": newScale,
    "photoOffsetX": newOffsetX,
    "photoOffsetY": newOffsetY,
    "photoUpdatedAt": FieldValue.serverTimestamp(),
    "updatedBy": userId
], merge: true)

// Listen for real-time changes
db.collection("site_config").document("discover_page").addSnapshotListener { snapshot, error in
    guard let data = snapshot?.data() else { return }
    // Update UI with new values
}
```

### Mentorship unlock → student_crm

The `mentorship_unlocked` localStorage flag should be migrated to the student's `student_crm` document:

```jsonc
// In student_crm collection
{
  "mentorshipUnlocked": true,
  "mentorshipUnlockedAt": "Timestamp"
}
```

Both platforms read this from the student's CRM record instead of local storage.

---

## Updated Action Items for iOS Team

1. **Add 7 new collections** to your iOS models: `student_crm`, `call_invitations`, `time_slot_config`, `time_slot_bookings`, `portal_messages`, `site_config`, `latex_templates`
2. **Add Schedule tab** to student portal — seat-booking UI with tier-based slots
3. **Add Inbox tab** to student portal — read messages from admin
4. **Read `call_invitations`** by student email to show the invitation banner
5. **Read `time_slot_config`** to populate the slot grid
6. **Write to `time_slot_bookings`** when a student confirms a booking
7. **Update `call_invitations` status** to `scheduled` when a slot is booked
8. **Read `portal_messages`** filtered by audience + user role/email
9. **Update `portal_messages.readBy`** when a student reads a message
10. **Respond to all open items above** — we need your answers to proceed with reconciliation
11. **Update tab count** to 28 (added `time-slot-config` and `messaging`)
12. **Read `site_config/discover_page`** for photo position — apply same zoom/pan on iOS Discover page
13. **Read `site_config/website_sections`** for section visibility — show/hide sections on iOS home
14. **Read `site_config/google_calendar`** for calendar connection status badge
15. **Read `site_config/roles`** for shared role definitions in user management
16. **Read `latex_templates` collection** for custom CV templates in CV builder
17. **Write to `site_config`** when admin adjusts any config on iOS (photo, sections, roles, calendar)
18. **Use `addSnapshotListener`** on `site_config` docs for real-time sync between platforms
19. **Coffee chat price**: Remove `originalPriceUSD: 17.5` from any hardcoded iOS seed data (set to 0 or remove)
20. **Mentorship unlock**: Read `mentorshipUnlocked` from `student_crm` doc instead of local storage

---

*This document supersedes `WEB_UPDATE_IOS_JULY10.md` and includes all information from it plus additional context.*

*Shared in `shared/docs/WEB_IOS_FULL_SYNC_JULY10.md`*
