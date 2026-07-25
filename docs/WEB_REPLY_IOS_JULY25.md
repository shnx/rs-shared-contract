# Web → iOS Reply — July 25, 2026

Answers to `IOS_REQUEST_TO_WEB_JULY25.md`.

---

## Q1: `paymentMethod` ambiguity

**Keep `paymentMethod` with the new gateway semantics.** The July 24 Part 1 Q4 statement to "drop it" was an error and is fixed now.

```swift
var paymentMethod: String?  // "tap" | "stripe" | "paypal" | "manual" | "free"
```

- `tap`     → `tapChargeId` populated
- `stripe`  → `stripeSessionId` populated
- `paypal`  → `paypalOrderId` populated
- `manual`  → admin marked paid outside gateway
- `free`    → 100% coupon (`discountType == "free"`)

**Canonical `Booking` fields:**
```swift
let id, userId?, email, name, company?, serviceOfferingId, serviceType: String
let scheduledAt, createdAt: Date
let durationMinutes: Int
let status: String                 // pending | confirmed | completed | cancelled
let priceUSD: Double
let stripeSessionId?, tapChargeId?, tapUrl?, paypalOrderId?, paypalTransactionId?: String
let sessionLink?, paymentMethod?, couponCode?, notes?: String
let paymentStatus: String          // unpaid | paid
```

**Web `Booking` interface (`src/types/index.ts`) updated to match.**

---

## Q2: `findBookingsByEmail` response shape

**Returns plain JSON, not Firestore `Timestamp` objects.** Callable CFs serialize Admin SDK data to JSON.

```json
{
  "success": true,
  "bookings": [
    {
      "id": "<documentId>",
      "email": "...",
      "name": "...",
      "scheduledAt": { "_seconds": 1719878400, "_nanoseconds": 0 },
      "createdAt":   { "_seconds": 1719878400, "_nanoseconds": 0 },
      "status": "confirmed",
      "paymentStatus": "paid",
      "priceUSD": 4.5
    }
  ]
}
```

- Key is `id` (document ID).
- Timestamps are `{ _seconds, _nanoseconds }` maps.
- Use `JSONDecoder`; `Firestore.Decoder` will not work.

```swift
struct FirestoreTimestamp: Codable {
    var _seconds: Int
    var _nanoseconds: Int
    var date: Date { Date(timeIntervalSince1970: TimeInterval(_seconds)) }
}
```

---

## Q3: Code verification endpoint

**✅ Deployed: `verifyBookingCode`.**

`sendBookingVerifyCode` now:
- Stores a 6-digit code per email for 10 minutes in `booking_verify_codes/{normalizedEmail}`.
- Still accepts a client `code` for backward compatibility (web does this).
- Generates a code if none is provided.

**`verifyBookingCode`:**
```swift
try await Functions.functions(region: "us-central1")
    .httpsCallable("verifyBookingCode")
    .call(["email": email, "code": code])
```

```json
{ "success": true, "verified": true }   // or false
```

**iOS flow:**
1. Call `sendBookingVerifyCode(["email": email])` (omit `code`).
2. User enters code from email.
3. Call `verifyBookingCode(["email": email, "code": code])`.
4. If `verified == true`, call `findBookingsByEmail`.

No `bookingId` required.

---

## Q4: Registration when `systemUsers` already exists

**Show a "Sign in" prompt.** Do not allow setting a new password and linking to an existing account without re-authentication.

- If `findUserByEmail(email)` returns a document: prompt sign-in.
- Offer password reset if they forgot it.
- Keep `registerFromBooking` throwing `emailAlreadyInUse`.

---

## Q5: `createPayPalOrder` iOS integration

**Creates a pending booking server-side and returns the PayPal approval URL.**

```json
// Request
{
  "bookingDetails": {
    "email": "...",
    "name": "...",
    "scheduledAt": "2026-07-30T09:00:00.000Z",
    "notes": "...",
    "finalPrice": 4.50
  }
}

// Response
{ "success": true, "url": "https://www.paypal.com/checkoutnow?token=...", "orderId": "..." }
```

- `url` is the PayPal approval page (open in `SFSafariViewController`).
- `orderId` is stored in `booking.paypalOrderId`.
- `paymentMethod` is set to `"paypal"`.
- `paypalTransactionId` is set later by `paypalIPN`.
- Return deep link: `robotics-sciences://book/confirm?booking_id=<id>&paypal=1` (custom URL scheme) or capture `SFSafariViewController` navigation to your success URL.
- Cancellation URL: `discover?cancelled=1`.

---

## Q6: Calendar sync and confirmation emails

**`syncBookingToCalendar` is not automatic.** Call it explicitly when the booking is confirmed.

- Free booking: call immediately after booking creation.
- Paid booking: call after `paymentStatus == "paid"`.
- Manual confirmation: call when status changes to `confirmed`.

```swift
try await Functions.functions(region: "us-central1")
    .httpsCallable("syncBookingToCalendar")
    .call(["bookingId": bookingId])
```

**Booking confirmation email:** No standalone `sendBookingConfirmation` CF exists yet. For iOS, either:
- Ask web to build `sendBookingConfirmation({ bookingId })` CF, or
- Send via iOS `EmailCenter`/`portal_messages` after confirmation.

---

## Q7: `slot_groups` and `slot_overrides` schemas

### `slot_groups`
```swift
struct SlotGroup: Codable {
    let id: String
    let name: String
    let nameAr: String?
    let slotTimes: [String]        // ["09:00","11:00","13:30"], Berlin time
    let maxSlotsPerDay: Int
    let activeDays: [Int]          // 0=Sun ... 6=Sat, empty = all
    let isActive: Bool
    let priority: Int
    let createdAt: Date
}
```

### `slot_overrides`
```swift
struct SlotOverride: Codable {
    let id: String
    let type: String               // "add" | "remove" | "block"
    let date: String               // "YYYY-MM-DD"
    let endDate: String?
    let startTime: String          // "HH:MM"
    let endTime: String            // "HH:MM"
    let label: String?
    let createdAt: Date
    let createdBy: String
}
```

### Linkage
- `coupons.slotGroup` → `slot_groups.id`
- `meeting_invitations.slotGroup` → `slot_groups.id`
- `slot_overrides` are applied globally to the calendar generation; no direct FK to a booking.

---

## Q8: Admin portal 4-category restructure

**Target structure (July 24):**

```
Dashboard (overview)
  - documents
  - timeline (accounting only)
  - course-overview (education/client)
  - riad-portal (real_estate only)

People
  - funnel-manager
  - user-360

Bookings & Services
  - calendar
  - invitations-coupons
  - service-offerings
  - session-outcomes
  - price-escalation

Settings
  - projects
  - user-management
  - admin-settings
  - link-manager
  - tap-config
```

Note: `tabRegistry.ts` still has 7 categories internally, but the sidebar groups them visually into the 4 headings above. iOS can implement the 4 groups immediately using the `category` field on each `TabDefinition`.

---

## Deployed Cloud Functions

- `findBookingsByEmail` ✅
- `sendBookingVerifyCode` ✅ (updated to generate/store codes)
- `verifyBookingCode` ✅ (new)
- `createPayPalOrder` ✅ (sets `paymentMethod` and `paypalOrderId`)

All are live in `us-central1`.

---

## Action items for iOS

1. Switch `findBookingsByEmail` decoding to `JSONDecoder`.
2. Use `verifyBookingCode` after `sendBookingVerifyCode`.
3. Keep `paymentMethod` and add `paypalOrderId`/`paypalTransactionId`.
4. Implement PayPal flow using `createPayPalOrder` + `SFSafariViewController` + custom URL scheme.
5. Call `syncBookingToCalendar` after booking confirmation.
6. Restructure `AdminPortalView` to 4 categories shown above.

---

*Pushed to `rs-shared-contract` `main` branch.*
