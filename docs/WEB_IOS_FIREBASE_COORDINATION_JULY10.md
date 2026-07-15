# Web ↔ iOS: Shared Firebase Data Coordination Plan — July 10

**From:** Web team (Mohammad Shannak)
**To:** iOS team
**Date:** July 10, 2026
**Re:** Ensuring both platforms read/write Firebase correctly — calendar management, website controls, ADY, and Riad Shannak

---

## Core Principle

**Firebase Firestore is the single source of truth for ALL shared data.** Both platforms must:

1. **Read** from Firestore collections on app launch
2. **Write** changes to Firestore (not just local storage)
3. **Listen** with `onSnapshot` for real-time updates when the other platform changes something
4. **Use the same field names** (camelCase, as documented below)

---

## 1. Shared Config — `site_config` Collection

Both platforms must read these documents and reflect changes in real-time.

### Document: `site_config/website_sections`

Controls which sections appear on the website/app.

```json
{
  "showNews": false,
  "showProjects": false,
  "showCourses": false,
  "showMembership": false,
  "showTutorials": false,
  "updatedAt": "2026-07-10T12:00:00Z",
  "updatedBy": "shannak"
}
```

**Web:** Reads on page load, writes when admin toggles sections in Website Manager.
**iOS:** Should read on app launch and show/hide corresponding tabs or sections.

### Document: `site_config/discover_page`

Controls the photo on the Discover page.

```json
{
  "photoPosition": {
    "scale": 1.2,
    "offsetX": 0,
    "offsetY": -50
  },
  "updatedAt": "2026-07-10T12:00:00Z"
}
```

**Web:** Admin can adjust zoom/pan in Discover page controls.
**iOS:** Read if you build a Discover page. Apply same transform to the photo.

### Document: `site_config/google_calendar`

Tracks whether Google Calendar is connected.

```json
{
  "connected": true,
  "connectedAt": "2026-07-10T12:00:00Z",
  "connectedBy": "shannak",
  "calendarId": "info@the-rs.com"
}
```

**Both platforms:** Read this to show a "Calendar Connected" badge. Write `connected: false` when disconnecting.

### Document: `site_config/roles`

Role definitions for user management.

```json
{
  "roles": [
    { "id": "admin", "name": "Admin", "permissions": ["*"] },
    { "id": "editor", "name": "Editor", "permissions": ["read", "write"] }
  ],
  "updatedAt": "2026-07-10T12:00:00Z"
}
```

**Both platforms:** Read for role-based access control. Write when admin creates/edits roles.

### Swift pattern for all site_config docs:

```swift
// Listen for real-time updates
db.collection("site_config").document("website_sections").addSnapshotListener { snapshot, error in
    guard let data = snapshot?.data() else { return }
    let showNews = data["showNews"] as? Bool ?? false
    let showProjects = data["showProjects"] as? Bool ?? false
    // Update UI
}

// Write changes
try await db.collection("site_config").document("website_sections").setData([
    "showNews": true,
    "showProjects": true,
    "updatedAt": FieldValue.serverTimestamp(),
    "updatedBy": "ios_user"
], merge: true)
```

---

## 2. Calendar Management — NEW Cloud Function

### Problem
Currently calendar sync requires OAuth token in localStorage. This only works on web. iOS can't create calendar events.

### Solution: `syncBookingToCalendar` Cloud Function

We built a server-side Cloud Function that creates Google Calendar events using a **service account** (no OAuth needed on client).

**Both platforms call it the same way:**

```swift
// iOS — using Firebase Functions
let function = Functions.functions()
function.httpsCallable("syncBookingToCalendar").call([
    "title": "Coffee Chat with John",
    "description": "15-min intro call",
    "startISO": "2026-07-15T10:00:00Z",
    "endISO": "2026-07-15T10:15:00Z",
    "attendeeEmail": "john@example.com",
    "attendeeName": "John"
]) { result, error in
    if let data = result?.data as? [String: Any] {
        let eventId = data["eventId"] as? String
        let htmlLink = data["htmlLink"] as? String
    }
}
```

```typescript
// Web — using Firebase Functions
import { syncBookingViaCloudFunction } from '../utils/googleCalendar';
const result = await syncBookingViaCloudFunction({
  title: 'Coffee Chat with John',
  description: '15-min intro call',
  startISO: '2026-07-15T10:00:00Z',
  endISO: '2026-07-15T10:15:00Z',
  attendeeEmail: 'john@example.com',
  attendeeName: 'John',
});
```

### Setup required (web team will do):
1. Create Google Service Account in Google Cloud Console
2. Enable Domain-Wide Delegation for Calendar API scope
3. Set Firestore secrets:
   - `GOOGLE_SERVICE_ACCOUNT_EMAIL`
   - `GOOGLE_SERVICE_ACCOUNT_PRIVATE_KEY`
   - `GOOGLE_CALENDAR_ID` (e.g. `info@the-rs.com`)
4. Deploy Cloud Function

### Calendar connection status
Both platforms should sync connection status to `site_config/google_calendar` (see above). When either platform connects or disconnects, write to this doc so the other platform knows.

### Booking flow (both platforms):
1. User books a session → creates doc in `bookings` collection
2. Call `syncBookingToCalendar` Cloud Function → creates Google Calendar event
3. Update booking doc with `calendarEventId` field
4. Send confirmation email via `sendBookingConfirmation` Cloud Function

---

## 3. ADY Project — Shared Collections

Both platforms read/write these. **No platform should use localStorage for ADY data.**

| Collection | Purpose | Web reads | Web writes | iOS reads | iOS writes |
|-----------|---------|-----------|-----------|-----------|-----------|
| `ady_transactions` | All financial transactions | ✅ | ✅ | ✅ | ✅ |
| `transactions` | Unprefixed fallback | ✅ (merged) | ✅ | ✅ (merged) | ❌ |
| `ady_periods` | Budget periods | ✅ | ✅ | ✅ | ✅ |
| `periods` | Unprefixed fallback | ✅ (merged) | ✅ | ✅ (merged) | ❌ |
| `ady_documents` | Documents | ✅ | ✅ | ✅ | ✅ |
| `documents` | Unprefixed fallback | ✅ (merged) | ✅ | ✅ (merged) | ❌ |
| `periodSummaries` | Period summaries | ✅ | ✅ | ✅ | ✅ |
| `ady_installments` | Payment installments | ✅ | ✅ | ✅ | ✅ |
| `ady_stores` | Store configs | ✅ | ✅ | — | — |
| `ady_crm` | CRM leads | ✅ | ✅ | ✅ | ✅ |

### Transaction fields both platforms must handle:

```
id: string
type: "payment" | "bill" | "check" | "cost" | "revenue" | "freelance" | "salary"
amount: number
currency: "JOD" | "USD" | "EUR"
date: Date
description: string
projectId: string
isRecurring: boolean (optional)
recurrenceFrequency: string (optional) — "weekly", "biweekly", "monthly", "quarterly", "yearly"
recurrenceEndDate: Date (optional)
recurrenceParentId: string (optional)
label: string (optional)
recipientName: string (optional)
source: string (optional) — "ios", "web", "email", "manual"
reviewStatus: string (optional) — "pending_review", "approved", "rejected"
emailInboxId: string (optional)
createdAt: Date
updatedAt: Date
```

**Important:** iOS treats `freelance` and `salary` as expenses (default fallback to `.cost`). Web does the same. **Both platforms must agree on this.**

### Recurring payments (iOS built this first):
- iOS Insights tab marks transactions as `isRecurring: true`
- Web will build matching Insights tab using same fields
- Both platforms should respect `isRecurring` flag and show recurring payments

---

## 4. Riad Shannak Project — Shared Collections

| Collection | Purpose | Web reads | Web writes | iOS reads | iOS writes |
|-----------|---------|-----------|-----------|-----------|-----------|
| `buildings` | Building/property data | ✅ | ✅ | ✅ | ✅ |
| `ady_transactions` | Property transactions (filtered by projectId) | ✅ | ✅ | ✅ | ✅ |
| `documents` | Property documents | ✅ | ✅ | ✅ | ✅ |
| `ady_documents` | Property documents (prefixed) | ✅ | ✅ | ✅ | ✅ |

### Building schema:

```
id: string
name: string
nameAr?: string
address?: string
addressAr?: string
floors?: number
units?: number
type?: string  // "residential", "commercial", "mixed"
projectId?: string
createdAt: Date
updatedAt: Date
```

**Both platforms:** When a building is created/updated on one side, the other should see it via Firestore listener.

---

## 5. Website Tools Project — Shared Collections

| Collection | Purpose | Web | iOS |
|-----------|---------|-----|-----|
| `job_submissions` | Job applications | ✅ Read/Write | ✅ Read/Write |
| `contact_submissions` | Contact form submissions | ✅ Read/Write | ✅ Read/Write |
| `courses` | Course catalog | ✅ Read/Write | ✅ Read |
| `enrollments` | Student enrollments | ✅ Read/Write | ✅ Read |
| `video_tutorials` | Video tutorials | ✅ Read/Write | ✅ Read |
| `video_assignments` | Assignments | ✅ Read/Write | ✅ Read |
| `student_memberships` | Membership records | ✅ Read/Write | ✅ Read |
| `career_opportunities` | Job board | ✅ Read/Write | ✅ Read |
| `crm_contacts` | CRM contacts | ✅ Read/Write | ✅ Read/Write |
| `ady_crm` | CRM leads | ✅ Read/Write | ✅ Read/Write |
| `student_crm` | Student CRM (mentorship unlock) | ✅ Read/Write | ✅ Read/Write |
| `latex_templates` | LaTeX CV templates | ✅ Read/Write | ✅ Read |
| `service_offerings` | Service packages/pricing | ✅ Read/Write | — |
| `availability` | Time slot availability | ✅ Read/Write | — |
| `bookings` | Session bookings | ✅ Read/Write | — |

### Mentorship unlock (NEW — both platforms must sync):

When a user pays for a mentorship coffee chat, the unlock flag is written to `student_crm`:

```json
{
  "email": "user@example.com",
  "name": "John",
  "mentorshipUnlocked": true,
  "mentorshipUnlockedAt": "2026-07-10T12:00:00Z"
}
```

**Both platforms:** Read `student_crm` by email to check if mentorship is unlocked. Write `mentorshipUnlocked: true` when payment is confirmed.

---

## 6. Collections iOS Uses That Web Doesn't (Yet)

| Collection | Purpose | Web plan |
|-----------|---------|----------|
| `work_sessions` | Employee time tracking | iOS-only, no web plan |
| `employee_permissions` | Employee access control | iOS-only, no web plan |

If web needs these later, iOS should document the schema.

---

## 7. Action Items for Both Teams

### Web team (will do):
1. ✅ Built `syncBookingToCalendar` Cloud Function (needs service account setup)
2. ✅ Fixed Discover page: success screen, email confirmation, calendar sync
3. ✅ Migrated mentorship unlock to `student_crm` in Firestore
4. ✅ Auto-seed availability slots if none exist
5. Set up Google Service Account + secrets for calendar Cloud Function
6. Build web Insights tab (matching iOS) using recurring payment fields
7. Add `onSnapshot` listeners to website config for real-time updates

### iOS team (should do):
1. Add `site_config` collection listeners — read `website_sections`, `google_calendar`, `roles`
2. Add `latex_templates` collection — read for CV builder if applicable
3. Read `student_crm` by email to check `mentorshipUnlocked` flag
4. Use `syncBookingToCalendar` Cloud Function for calendar events (no OAuth needed)
5. Write calendar connection status to `site_config/google_calendar`
6. Confirm `buildings` collection schema matches between platforms
7. Use `onSnapshot` on shared collections for real-time sync

### Both teams (coordinate):
1. **Agree on transaction type handling** — `freelance`/`salary` = expense on both sides ✅
2. **Agree on `timeline` collection name** — web renamed from `timeline_entries` to match iOS ✅
3. **Agree on Gemini model** — both on `gemini-2.5-flash` ✅
4. **Agree on booking flow** — create booking → sync calendar → send email
5. **Agree on `source` field** — use `"ios"` or `"web"` on all created records
6. **Agree on recurring payment fields** — use existing schema, no new fields

---

## 8. Firestore Rules (current)

```javascript
// site_config — both platforms need read/write (authenticated)
match /site_config/{docId} {
  allow read, write: if request.auth != null;
}

// latex_templates — both platforms need read (authenticated)
match /latex_templates/{docId} {
  allow read: if request.auth != null;
  allow write: if request.auth != null;
}

// student_crm — both platforms need read/write (authenticated)
match /student_crm/{docId} {
  allow read, write: if request.auth != null;
}

// All other collections — authenticated users
match /{document=**} {
  allow read, write: if request.auth != null;
}
```

---

*Reply by adding a new `.md` file to `shared/docs/` or comment on GitHub issue #1.*
