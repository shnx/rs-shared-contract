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

---

## Part 6: Answers to 16 New iOS Questions (from IOS_QUESTIONS_JULY24.md)

### Q1: Is PayPal the canonical payment method now?

**Answer:** **No single canonical provider.** Three active payment options:
- **Tap** — Jordan / MENA (Mada, Benefit, KNET, etc.)
- **Stripe** — Global USD credit cards
- **PayPal** — Global alternative

The DiscoverPage currently renders **PayPal Hosted Buttons** because PayPal is the fastest path for web, but Tap and Stripe remain fully active. iOS should offer all three (admin-configurable toggle).

---

### Q2: PayPal Hosted Buttons vs PayPal API Orders for iOS?

**Answer:** For iOS, use the **PayPal REST API / Orders v2** directly, or better — call our Cloud Function `createPayPalOrder` and capture via `paypalIPN` webhook.

**Recommended iOS flow:**
1. Create booking in Firestore (status: `pending`, paymentStatus: `unpaid`)
2. Call `createPayPalOrder` CF with `bookingId`, `amount`, `currency`, `description`
3. CF returns `orderId` and `approveUrl`
4. Open `approveUrl` in `SFSafariViewController`
5. PayPal redirects to `SITE_URL/discover?paypal=1&booking_id={bookingId}`
6. `paypalIPN` webhook updates booking to `paid`
7. iOS listens to booking doc for `paymentStatus == "paid"`

Do NOT use PayPal Hosted Buttons in iOS — that's a web SDK convenience.

---

### Q3: Should iOS keep the Tap admin config?

**Answer:** **Yes.** Tap is the primary MENA gateway. Keep Tap config in iOS admin payments settings. It is still used and maintained.

---

### Q4: Is Stripe deprecated?

**Answer:** **No.** Stripe is active for global USD cards. The comment in `functions/src/index.ts` about Stripe secret key being "commented out" refers to Firebase Secret Manager; the function now reads `stripe_config` from Firestore. iOS should keep Stripe as an option.

---

### Q5: Should `resumeBookingPayment` use PayPal instead of Stripe?

**Answer:** **No, keep it as-is.** `resumeBookingPayment` creates a new Stripe checkout for an existing unpaid booking. If the original booking was PayPal, create a new PayPal order instead. The function should be gateway-agnostic per original payment method. iOS should pick the right CF based on original booking fields (`tapChargeId`, `stripeSessionId`, or missing).

**Rule:**
- Original had `tapChargeId` → call `createTapCheckout`
- Original had `stripeSessionId` → call `resumeBookingPayment`
- Original had PayPal order → call `createPayPalOrder`

---

### Q6: Add `paypalOrderId` to `Booking` schema?

**Answer:** **Yes, add it.** Also fix the `paypalIPN` CF reusing `tapChargeId` — that was a shortcut. Correct fields:

```swift
struct Booking: Codable {
    // ...existing fields...
    var tapChargeId: String?
    var tapUrl: String?
    var stripeSessionId: String?
    var paypalOrderId: String?        // NEW — add this
    var paypalTransactionId: String?  // NEW — captured from IPN
    var paymentMethod: String?        // "tap" | "stripe" | "paypal" — recommended
    var sessionLink: String?
}
```

**Web will update:** `paypalIPN` will write `paypalOrderId` and `paypalTransactionId` instead of `tapChargeId`. This is a backend fix; no iOS action beyond adding the fields.

---

### Q7: Email exists detection — `findUserByEmail` vs `fetchSignInMethods`?

**Answer:** Use **`findUserByEmail` querying `systemUsers` by email.** Do NOT rely on `Auth.auth().fetchSignInMethods(forEmail:)` because:
- Invited users exist in `systemUsers` before they create Firebase Auth accounts
- Legacy users may have `systemUsers` docs without Firebase Auth yet
- `systemUsers` is the source of truth for portal profiles

```swift
func findUserByEmail(_ email: String) async -> SystemUser? {
    let snap = try? await db.collection("systemUsers")
        .whereField("email", isEqualTo: email.lowercased())
        .limit(to: 1)
        .getDocuments()
    return snap?.documents.first.flatMap { try? $0.data(as: SystemUser.self) }
}
```

---

### Q8: Should iOS add email verification for booking search?

**Answer:** **Yes.** For public (unauthenticated) users searching by email, require a 6-digit code via `sendBookingVerifyCode` before allowing destructive actions (cancel, reschedule). Direct booking ID link is trusted and does not need verification. Logged-in users whose email matches the booking are also trusted.

---

### Q9: Email-only search vs name + email?

**Answer:** **Align to email-only or booking ID.** Firestore `bookings` rules restrict `list` to auth users; public search needs to be email-only (or use the Cloud Function we will build, see Q16). Name filtering is not supported by Firestore and is unnecessary once email verification is in place.

---

### Q10: Should iOS add project selector?

**Answer:** **Yes.** Web portal requires selecting a project before showing tabs. iOS should add a project selector (dropdown or initial screen). This drives tab visibility and permissions. Use `project_members` to determine which projects the user can access.

---

### Q11: Per-project tab visibility for iOS?

**Answer:** **Yes, implement it.** Use `project.type` to filter tabs:
- `accounting` → Dashboard, Documents, Timeline, Finance tabs
- `website` → Bookings & Services, People, Website Manager tabs
- `real_estate` → Riad Portal, Buildings tabs

Admins (`role == "admin"`) can see all tabs; other roles see tabs based on `permissions`.

---

### Q12: Coupon `discountType == "fixed"` — `discountValue` in cents?

**Answer:** **No.** `discountValue` for `fixed` is in **cents of the currency** (e.g. `100` = `$1.00` or `1.0 JOD`). This is the convention. Wait, earlier we said 1 = $0.01. Confirming: **100 = $1.00 USD**. For UI display, divide by 100.

```swift
let price = Double(discountValue) / 100.0
```

---

### Q13: `meeting_invitations` vs `meetingInvitations` collection name?

**Answer:** **Firestore collection is `meeting_invitations`** (snake_case). `meetingInvitations` is the TypeScript variable name in `src/utils/database.ts`. iOS should use `meeting_invitations` as the collection path.

---

### Q14: Should iOS implement `slot_overrides`?

**Answer:** **Yes, if you are building admin calendar.** `slot_overrides` allows one-off blocking or adding slots per date (e.g. blocked holiday, extra Saturday session). For public booking flow, iOS only needs to read them and merge with recurring availability. For admin, add CRUD. Priority: **P2**.

---

### Q15: Use `syncBookingToCalendar` CF exclusively?

**Answer:** **Yes.** Use `syncBookingToCalendar` CF exclusively. No client-side Google OAuth needed. It is server-side with Google Service Account, no auth required. Call it for both free and paid bookings after confirmation.

---

### Q16: How to search bookings for unauthenticated users? (CRITICAL)

**Answer:** This is a real issue. Current Firestore rules require `request.auth != null` for `list` on `bookings`, so unauthenticated `whereField("email", isEqualTo:)` will fail.

**Web will fix by building a Cloud Function `findBookingsByEmail`:**
```swift
try await Functions.functions(region: "us-central1")
    .httpsCallable("findBookingsByEmail")
    .call(["email": email])
// Returns: [Booking] (array of user's bookings), no auth required
```

**Why not change Firestore rules?** Allowing public `list` by email would expose booking IDs and names to anyone who guesses an email. The CF will also rate-limit and require email verification code before returning results (optional but recommended).

**iOS action:** Use `findBookingsByEmail` CF for public booking search. For logged-in users, use direct Firestore query `bookings` by `userId` or `email`.

**ETA:** Web will deploy `findBookingsByEmail` within 24h. Until then, iOS can test with a hardcoded auth user or use a test collection.

---

## Part 7: What iOS Team Got Wrong in Their July 24 Status Update

1. **"No new web answer file exists"** — Incorrect. `shared/docs/WEB_REPLY_IOS_JULY24.md` exists and was pushed. It answers the 10 pending July 15 questions and the new User 360 / booking UI changes.
2. **"Latest web answers are July 9 / July 2"** — Outdated. The latest are `WEB_REPLY_IOS_JULY24.md` and `IOS_CHANGELOG_JULY24.md`.
3. **"WEB_RESPONSE_IOS_JULY8 says Tap only, but DiscoverPage uses PayPal = conflict"** — July 8 doc is old. The July 24 docs and code clearly support Tap + Stripe + PayPal. No conflict — the system evolved.
4. **"10 questions from July 15 still open"** — They were answered in `WEB_REPLY_IOS_JULY24.md` Part 1.
5. **"Need `findBookingsByEmail` CF"** — Correct. This is the only genuinely new critical issue. Web will build it.

---

## Part 8: Updated iOS Action Items (After July 24 Round 2)

### P1 (Must Do)
1. Switch public booking search to `findBookingsByEmail` CF (when deployed)
2. Add `paypalOrderId` and `paymentMethod` fields to `Booking` model
3. Use `createPayPalOrder` CF for PayPal iOS flow
4. Add `sendBookingVerifyCode` for public booking management

### P2 (Should Do)
5. Build User 360° profile page
6. Add project selector to admin portal
7. Implement per-project tab visibility
8. Add `SlotOverride` model + repository
9. Add `slot_groups` and `site_config/booking_settings` integration
10. Restructure `AdminPortalView` to 4-category tab bar

### P3 (Nice to Have)
11. Add PayPal / Stripe / Tap payment method toggle in admin payments settings
12. Add `meeting_invitations` (snake_case) collection path update
