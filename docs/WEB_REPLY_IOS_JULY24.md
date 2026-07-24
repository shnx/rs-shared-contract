# Web → iOS Team — Complete Q&A Update

**Date:** July 24, 2026 (evening)
**From:** Web team
**To:** iOS team
**Re:** Answers to all open iOS questions + latest changes (User 360, booking UI, nav changes, slot groups, coupons)

---

## Part 1: Original 10 Questions (from IOS_WEB_SYNC_JULY15.md)

All 10 were initially answered in `IOS_CHANGELOG_JULY20.md` section 8. Here are the **updated** answers reflecting the July 24 portal redesign:

### Q1: Availability schema — recurring weekly vs concrete slots?

**Answer:** ✅ **Recurring weekly is canonical.** Confirmed and deployed.

```swift
struct Availability: Codable {
    let id: String
    let dayOfWeek: Int       // 0=Sunday..6=Saturday
    let startTime: String    // "10:00" 24h format
    let endTime: String      // "18:00"
    let bufferMinutes: Int   // default 15
    let maxBookingsPerDay: Int
    let priceTier: String?   // free|discounted|standard|premium
}
```

**iOS action:** Stop writing concrete slot docs (`startDate`/`endDate`). Use recurring weekly model. Generate slots client-side from the weekly window + `site_config/booking_settings` for hour range and interval.

**Who writes:** Both platforms can write availability docs. Web admin is primary. iOS admin can also manage.

**`isBooked`/`bookedBy`:** These live on the `bookings` collection, NOT on `availability` docs. An availability slot is "booked" when a `bookings` doc exists with matching date+time and status ≠ cancelled.

---

### Q2: CRM collection naming — `ady_crm` vs `crm_contacts`?

**Answer:** Keep `ady_crm` on iOS for now. Web uses `crm_contacts` (unprefixed) and `student_crm` for student-specific CRM. Migration is planned but not yet scheduled. **No action needed now.**

---

### Q3: `service_offerings` schema — should iOS expand?

**Answer:** ✅ **Yes, expand.** Web canonical schema includes:

```swift
struct ServiceOffering: Codable {
    let id: String
    let name: String
    let nameAr: String?
    let description: String
    let descriptionAr: String?
    let type: String          // coffee_time, career_roadmap, interview_success, portfolio_builder, company_assessment, custom_training, advisory_retainer, membership, course_access
    let audience: String      // "student" | "company"
    let category: String      // "career", "training", "advisory", "technical"
    let priceUSD: Double
    let originalPriceUSD: Double?
    let durationMinutes: Int
    let isActive: Bool
    let outcome: String?
    let outcomeAr: String?
    let deliverables: [String]?
    let deliverablesAr: [String]?
    let duration: String?     // "4 weeks", "Ongoing"
    let commitment: String?   // "4 sessions", "Monthly meetings"
    let pricePer: String?     // "package", "month", "hour"
    let featured: Bool?
    let order: Int?
    let createdAt: Date
}
```

**iOS action:** Expand model to include all fields. See `src/types/index.ts:794-820` for the TypeScript source of truth.

---

### Q4: `bookings` schema — align to canonical?

**Answer:** ✅ **Yes, align.** Drop `consultationId`, `tier`, `priceEUR`, `paymentMethod`, `invoiceUrl`. The canonical booking schema is:

```swift
struct Booking: Codable {
    let id: String
    let userId: String?
    let email: String
    let name: String
    let company: String?
    let serviceOfferingId: String
    let serviceType: String
    let scheduledAt: Date
    let durationMinutes: Int
    let status: String         // pending, confirmed, completed, cancelled
    let paymentStatus: String  // unpaid, paid
    let priceUSD: Double
    let couponCode: String?    // NEW (July 24) — optional, shows which coupon was used
    let tapChargeId: String?
    let stripeSessionId: String?
    let notes: String?
    let createdAt: Date
}
```

**New field (July 24):** `couponCode: String?` — optional. Web now writes this when a coupon is applied. iOS can read it to display which coupon was used per booking. No migration needed — field is optional/additive.

---

### Q5: Pricing currency — USD only or JOD+USD?

**Answer:** **Two separate systems, keep separate.**
- **Web bookings** (`bookings` collection): USD only (`priceUSD`). Price source: `site_config/booking_settings.bookingFixedPrice` (default 4.5 USD).
- **iOS slot bookings** (`time_slot_bookings` collection): JOD. This is a separate system for in-person slot booking at the office.

Do NOT unify them. They serve different purposes.

---

### Q6: `timeline` vs `timeline_entries`?

**Answer:** ✅ **Use `timeline`.** Web has renamed to `timeline`. Update `TimelineRepository` collection path from `timeline_entries` to `timeline`.

---

### Q7: `syncBookingToCalendar` deployed?

**Answer:** ✅ **Yes, deployed and working.** Server-side only (Google Service Account, no client OAuth). Call it freely — no auth required.

```swift
try await Functions.functions(region: "us-central1")
    .httpsCallable("syncBookingToCalendar")
    .call([
        "title": "Discovery Call — Ahmad Ali",
        "description": "Booking confirmed. Email: ahmad@example.com",
        "startISO": "2026-07-25T10:00:00.000Z",
        "endISO": "2026-07-25T10:30:00.000Z",
        "attendeeEmail": "ahmad@example.com",
        "attendeeName": "Ahmad Ali"
    ])
```

**Important:** Call this for BOTH free and paid bookings. Free bookings must also sync to calendar + send confirmation email.

---

### Q8: Stripe vs Tap?

**Answer:** **Both active. Three payment providers:**
- **Tap** (primary for Jordan/Middle East) — `createTapCheckout` CF
- **Stripe** (global USD) — `createStripeCheckout` CF
- **PayPal** (third option) — `createPayPalOrder` CF

iOS can use any/all. Tap is recommended for MENA users. Stripe for international.

---

### Q9: Duplicate "First Payment" period?

**Answer:** Web will delete `1764518075230bwtmp8tlg` from `ady_periods` separately. If iOS still sees it, filter it out client-side by ID or wait for the deletion.

---

### Q10: `updateEnrollmentCount` Cloud Function?

**Answer:** **Not built.** Update `course.studentCount` directly in Firestore for now:
```swift
try await db.collection("courses").document(courseId).updateData([
    "studentCount": FieldValue.increment(Int64(1))
])
```

---

## Part 2: iOS Control Parity Requests (from IOS_WEB_SYNC_JULY15.md section 3)

| # | Request | Status | Notes |
|---|---------|--------|-------|
| 3a | Service Offerings CRUD on iOS | ✅ **Approved** | iOS can add full CRUD. Firestore rules allow auth writes. |
| 3b | Booking management (confirm/cancel/complete) on iOS | ✅ **Approved** | iOS can add booking management. Update `bookings.status` / `bookings.paymentStatus`. |
| 3c | CRM collection consolidation | ⏳ **Deferred** | Keep `ady_crm` on iOS for now. |
| 3d | Email Inbox approve/reject on iOS | ✅ **Approved** | iOS can add write to `email_inbox`. |
| 3e | Contact Submissions read + reply on iOS | ✅ **Approved** | iOS can read `contact_submissions` and reply via `sendContactReply` CF. |
| 3f | Membership Plans read on iOS | ✅ **Approved** | iOS can read `membership_plans` / `student_memberships`. |
| 3g | Course management on iOS | ✅ **Approved** | iOS can add toggle `isPublished`, edit title/description. |
| 3h | Video Tutorials CRUD on iOS | ✅ **Approved** | iOS can add upload/edit/toggle. |
| 3i | Tap Config read-only on iOS | ✅ **Approved** | iOS can read `tap_config` for status display. |
| 3j | Audit Log read on iOS | ✅ **Approved** | iOS can read `audit_log`. |
| 3k | Widget Configs read on iOS | ✅ **Approved** | iOS can read `widget_configs`. |
| 3l | Project Members CRUD on iOS | ✅ **Approved** | iOS can assign/remove members in `project_members`. |

**Firestore rules:** All the above collections already allow Firebase Auth reads/writes for authenticated users. No rules changes needed.

---

## Part 3: New Changes Since Last Sync (July 24, 2026)

### 3a. Portal Category Restructure (4 categories)

Web portal reorganized from 6 → 4 categories. iOS should match:

```
Dashboard:     Documents, Analytics
People:        Funnel Manager, User Management, Communication, User Messages
Bookings & Services: Calendar & Bookings, Invitations & Coupons, Service Offerings, Outcomes, Price Escalation, Proposals, Website Manager
Settings:      Projects, Payments, Admin Settings, Link Manager
```

iOS tab bar: 4 tabs with SF Symbols:
- Dashboard → `square.grid.2x2`
- People → `person.2`
- Bookings & Services → `calendar`
- Settings → `gearshape`

See `IOS_CHANGELOG_JULY24.md` sections 1-8 for full details, per-project visibility matrix, and role-based filtering.

---

### 3b. Slot Groups (NEW collection — `slot_groups`)

**Purpose:** Divide users into groups. Each group sees different time slots (scarcity effect).

```swift
struct SlotGroup: Codable, Identifiable {
    @DocumentID var id: String?
    var name: String
    var nameAr: String?
    var slotTimes: [String]       // ["09:00", "11:00", "13:30", "16:30"]
    var maxSlotsPerDay: Int       // 3 or 4
    var activeDays: [Int]         // [] = all days, [0,1,2,3,4] = Sun–Thu
    var isActive: Bool
    var priority: Int
    @ServerTimestamp var createdAt: Date?
}
```

**iOS implementation:**
1. Load active slot groups on booking screen
2. When user enters coupon, check `coupon.slotGroup`
3. If set, fetch slot group, filter displayed slots to `group.slotTimes`
4. If count > `maxSlotsPerDay`, deterministically pick subset (seed = date + coupon code)
5. If `activeDays` non-empty, only show slots on those days

Full Swift implementation guide: `IOS_CHANGELOG_JULY24.md` section 10 (steps 1-6).

**Firestore rules:** `slot_groups` — public read, portal auth for create/update, Firebase Auth for delete.

---

### 3c. Coupon Enhancements

Coupons now support:
- `slotGroup: String?` — link coupon to a slot group
- Activate/deactivate toggle (`isActive`)
- Edit after creation (discount type, value, maxUses, expiry, notes, slotGroup)
- Reset usage count (`usedCount` → 0)
- Status badges: Active (green), Inactive (gray), Expired (red), Fully Used (amber)
- Duplicate code prevention on create

**iOS action:** Mirror these features in coupon management UI.

---

### 3d. Booking Calendar Config (dynamic, not hardcoded)

All booking calendar settings are in Firestore `site_config/booking_settings`:

```swift
struct BookingCalendarConfig: Codable {
    var bookingMaxDate: String          // "2026-08-10" — max bookable date
    var bookingBlockStart: String       // ISO date — start of blocked period
    var bookingBlockEnd: String         // ISO date — end of blocked period
    var bookingAvailableDays: [Int]     // [0,1,2,3,4] = Sun–Thu
    var bookingBerlinHourStart: Int     // 9 (9am Berlin)
    var bookingBerlinHourEnd: Int       // 18 (6pm Berlin)
    var bookingSlotIntervalMinutes: Int // 30
    var bookingFixedPrice: Double       // 4.5 (USD)
}
```

**iOS action:** Read this dynamically. Do NOT hardcode 9am-6pm or any values. Admin can change them anytime via Calendar Management settings.

---

### 3e. New Cloud Function

| Function | Params | Purpose |
|---|---|---|
| `sendBookingVerifyCode` | `email, bookingId` | Sends 6-digit verification code email for booking ownership verification (used in public booking cancellation flow) |

---

### 3f. User 360° Profile Page (Admin Only — NEW July 24 evening)

**What:** New admin tab `user-360` that aggregates ALL user data in one unified view. High-value feature for iOS to build.

**Tabs:** Overview, Timeline, CV (inline preview + AI analysis), Bookings, Emails (send + history), Messages, Account (permissions), Analytics (sessions), Portal Preview (impersonation).

**Data sources:** submissions, bookings, CRM contacts, system users, call invitations, portal messages, analytics events/sessions, email logs, service offerings.

**iOS recommendation:** **Build this.** It's a great admin feature. Here's how:

```swift
struct User360Profile {
    let systemUser: SystemUser?           // systemUsers (by email)
    let submissions: [JobSubmission]      // job_submissions (by email)
    let bookings: [Booking]               // bookings (by email)
    let crmContacts: [CRMContact]         // crm_contacts (by email)
    let studentCRM: [StudentCRMContact]   // student_crm (by email)
    let callInvitations: [CallInvitation] // call_invitations (by studentEmail)
    let messages: [PortalMessage]         // portal_messages (by userId)
    let emailLogs: [EmailLog]             // email_logs (by to/email)
    let analyticsEvents: [AnalyticsEvent] // analytics_events
    let analyticsSessions: [VisitorSession] // analytics_sessions
}

// Fetch all user data by email — parallel async calls
func loadUser360(email: String) async throws -> User360Profile {
    async let user = fetchSystemUser(email: email)
    async let subs = fetchSubmissions(email: email)
    async let books = fetchBookings(email: email)
    async let crm = fetchCRMContacts(email: email)
    async let invites = fetchCallInvitations(email: email)
    async let logs = fetchEmailLogs(email: email)
    // ...return User360Profile(...)
}
```

**iOS screens:**
- `User360View.swift` — SwiftUI TabView with tabs: Overview, Timeline, CV, Bookings, Emails, Messages, Account, Analytics
- Tappable user names/emails in Funnel Manager, User Management, Booking Management → navigate to User360
- Email sending via `sendSubmissionResponse` CF
- CV preview via `extractCVFromBase64` CF or inline PDF viewer
- Analytics sessions timeline (pages visited, device, geo)

**Firestore queries (all by email):**
```swift
db.collection("job_submissions").whereField("email", isEqualTo: email)
db.collection("bookings").whereField("email", isEqualTo: email)
db.collection("crm_contacts").whereField("email", isEqualTo: email)
db.collection("call_invitations").whereField("studentEmail", isEqualTo: email)
db.collection("email_logs").whereField("to", isEqualTo: email)
db.collection("portal_messages").whereField("userId", isEqualTo: userId)
db.collection("systemUsers").whereField("email", isEqualTo: email)
```

**No new collections needed** — reads existing collections only. All queries by email (or userId for messages). Firestore rules already allow auth reads.

---

### 3g. Booking Management Enhancements (Admin Only — July 24 evening)

- **Payment & Coupon column**: Shows `couponCode` badge, payment method (PayPal/Tap/Stripe), payment status detail
- **Pending analysis panel**: Shows stuck reasons (payment incomplete, time elapsed, coupon used)
- **Email Center modal**: Live HTML preview before sending — templates: Reconfirmation, Welcome, Apology, Custom
- **New field on bookings**: `couponCode?: String` — optional, additive

**iOS impact:** The `couponCode` field on bookings is now available in Firestore. iOS can read it to display which coupon was used per booking. No schema migration needed.

---

### 3h. Public Website Navigation Changes (July 24 evening)

- **Removed:** "My Bookings" tab from top navigation bar and footer
- **Kept:** `/bookings` and `/my-bookings` routes still work — accessible via Help page
- **Reason:** Cleaner public nav

**iOS impact:** **None.** iOS app has its own navigation. Web nav changes don't affect the app.

---

## Part 4: Summary — iOS Action Items (Updated July 24 evening)

### Must Do (P1)
1. ✅ Call `syncBookingToCalendar` after booking confirmation (free + paid)
2. ✅ Call `sendBookingConfirmation` after booking confirmation
3. ✅ Adopt recurring weekly availability schema
4. ✅ Update `timeline` collection name (from `timeline_entries`)
5. ✅ Expand `ServiceOffering` model to match canonical schema
6. ✅ Align `Booking` model to canonical schema (add `couponCode?`, drop `consultationId`, `tier`, `priceEUR`, `paymentMethod`, `invoiceUrl`)
7. ✅ Restructure iOS tab bar to 4 categories matching web
8. ✅ Add Slot Groups management to Calendar & Bookings
9. ✅ Update Coupon management with new features (toggle, edit, reset, slot group, status badges)
10. ✅ Read booking calendar config dynamically from `site_config/booking_settings`

### Should Do (P2)
11. **Build User 360° profile page** — unified user view with tabs (Overview, CV, Bookings, Emails, Messages, Analytics). Tappable names in Funnel Manager/User Management/Bookings → User360. See section 3f above.
12. Add Booking Invitations screen (select students, send emails)
13. Add auto-welcome toggle for submissions
14. Add user management CRUD with permission toggles
15. Add availability management UI (day/time/buffer/max/tier)
16. Add calendar blocks management
17. Read `call_invitations` by email for student portal banner
18. Add `sendBookingVerifyCode` CF call for booking cancellation verification
19. Read `couponCode` from bookings to display coupon used

### Nice to Have (P3)
20. Add send invitation/CV feedback buttons in user list
21. Read `site_config/roles` for custom roles
22. Add quotation builder
23. Add contact submissions reply
24. Add Link Manager to Settings tab
25. Add Proposals screen to Bookings & Services tab

### No Action Needed (web-only)
- Booking Management UI enhancements — admin web tool (but `couponCode` field is available for iOS to read)
- "My Bookings" removal from public nav — web-only change
- Email Center modal — admin web tool

---

## Part 5: Admin-Only Guarantee

All July 24 evening changes are **admin portal tools**. None affect:
- Public website pages (Discover, Courses, Membership, etc.)
- Candidate/Student portal experience
- Client portal experience
- Firestore schemas or security rules (except `couponCode` field which is optional/additive)

**iOS recommendation:** User 360° profile page is recommended for iOS implementation (see section 3f). Booking Management `couponCode` field is available for iOS to read. All other evening changes are web-only.

---

*Shared in `shared/docs/WEB_REPLY_IOS_JULY24.md`*
*See also: `docs/IOS_CHANGELOG_JULY24.md` for full technical details*
