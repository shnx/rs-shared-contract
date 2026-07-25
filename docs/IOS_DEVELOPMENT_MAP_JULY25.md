# iOS Development Map — July 25, 2026

This document provides a comprehensive map of the iOS development state, applied fixes, and roadmap based on web team responses.

---

## 1. Applied Fixes (July 24–25, 2026)

### 1.1 Public Booking Search
- **File:** `PublicSiteView.swift`
- **Change:** Replaced client-side `searchByNameAndEmail` with `findBookingsByEmail` Cloud Function
- **Status:** ✅ Completed
- **Details:** Users now search by email only; Cloud Function returns matching bookings

### 1.2 Booking Model Alignment
- **File:** `Models.swift`
- **Changes:**
  - Removed: `consultationId`, `tier`, `priceEUR`, `invoiceUrl`
  - Added: `couponCode`, `tapChargeId`, `tapUrl`, `paypalOrderId`, `paypalTransactionId`, `sessionLink`, `paymentMethod`
  - Updated `paymentMethod` values: `"tap" | "stripe" | "paypal" | "manual" | "free"`
- **Status:** ✅ Completed

### 1.3 BookingFull and Coupon Models
- **File:** `AdminModels.swift`
- **Changes:**
  - `Coupon`: Added `slotGroup`, updated `discountType` to `free/percent/fixed`
  - `BookingFull`: Added same payment fields as `Booking`
  - Updated `paymentMethod` comment to include `manual` and `free`
- **Status:** ✅ Completed

### 1.4 AuthManager Registration Pre-check
- **File:** `AuthManager.swift`
- **Change:** Added `findUserByEmail` method to check `systemUsers` before Firebase Auth creation
- **Status:** ✅ Completed
- **UX:** Throws `emailAlreadyInUse` if user exists; prompts sign-in

### 1.5 Cloud Function Integration
- **File:** `FirestoreRepository.swift`
- **Added:**
  - `FirestoreTimestamp` helper for decoding CF JSON responses
  - `findBookingsByEmail` (uses `JSONDecoder` with custom date strategy)
  - `sendBookingVerifyCode`
  - `verifyBookingCode` (new)
  - `syncBookingToCalendar`
  - `createPayPalOrder`
- **Status:** ✅ Completed

### 1.6 PayPal Payment Flow
- **File:** `PayPalPaymentView.swift` (new)
- **Implementation:**
  - Calls `createPayPalOrder` CF
  - Opens PayPal URL in `SFSafariViewController`
  - Handles deep link callback via custom URL scheme `robotics-sciences://book/confirm?booking_id=...&paypal=1`
- **File:** `rsApp.swift`
- **Added:** `onOpenURL` handler to process PayPal deep link
- **Status:** ✅ Completed

### 1.7 Slot Models
- **File:** `AdminModels.swift`
- **Added:**
  - `SlotGroup`: `id`, `name`, `nameAr`, `slotTimes`, `maxSlotsPerDay`, `activeDays`, `isActive`, `priority`, `createdAt`
  - `SlotOverride`: `id`, `type`, `date`, `endDate`, `startTime`, `endTime`, `label`, `createdAt`, `createdBy`
- **Status:** ✅ Completed

### 1.8 Admin Portal Restructure
- **File:** `AdminPortalView.swift`
- **Change:** Restructured to 4 categories with expandable sub-items:
  - **Dashboard:** Documents, Timeline, Course Overview, Riad Portal
  - **People:** Funnel Manager, User 360
  - **Bookings & Services:** Calendar, Invitations & Coupons, Service Offerings, Session Outcomes, Price Escalation
  - **Settings:** Projects, User Management, Admin Settings, Link Manager, Tap Config
- **Status:** ✅ Completed

---

## 2. Cloud Functions Deployed (us-central1)

| Function | Purpose | iOS Integration |
|----------|---------|-----------------|
| `findBookingsByEmail` | Search bookings by email | `BookingsRepository.findBookingsByEmail` |
| `sendBookingVerifyCode` | Send 6-digit verification code | `BookingsRepository.sendBookingVerifyCode` |
| `verifyBookingCode` | Verify code before destructive action | `BookingsRepository.verifyBookingCode` |
| `createPayPalOrder` | Create PayPal order, return approval URL | `BookingsRepository.createPayPalOrder` |
| `syncBookingToCalendar` | Sync booking to admin calendar | `BookingsRepository.syncBookingToCalendar` |

---

## 3. Data Models (Canonical Schema)

### 3.1 Booking
```swift
struct Booking: Codable, Identifiable {
    @DocumentID var id: String?
    var userId: String?
    var email: String
    var name: String
    var company: String?
    var serviceOfferingId: String?
    var serviceType: String?
    var scheduledAt: Date?
    var durationMinutes: Int?
    var status: String?              // pending | confirmed | completed | cancelled
    var paymentStatus: String?       // unpaid | paid
    var priceUSD: Double?
    var couponCode: String?
    var tapChargeId: String?
    var tapUrl: String?
    var stripeSessionId: String?
    var paypalOrderId: String?
    var paypalTransactionId: String?
    var paymentMethod: String?       // tap | stripe | paypal | manual | free
    var sessionLink: String?
    var notes: String?
    var createdAt: Date?
}
```

### 3.2 Coupon
```swift
struct Coupon: Codable, Identifiable {
    @DocumentID var id: String?
    var code: String
    var discountType: String?         // free | percent | fixed
    var discountValue: Double?       // free=100, percent=1-99, fixed=cents
    var maxUses: Int?
    var usedCount: Int?
    var slotGroup: String?
    var expiresAt: Date?
    var isActive: Bool?
    var notes: String?
    var createdAt: Date?
}
```

### 3.3 SlotGroup
```swift
struct SlotGroup: Codable, Identifiable {
    let id: String
    let name: String
    let nameAr: String?
    let slotTimes: [String]          // ["09:00","11:00","13:30"], Berlin time
    let maxSlotsPerDay: Int
    let activeDays: [Int]            // 0=Sun ... 6=Sat, empty = all
    let isActive: Bool
    let priority: Int
    let createdAt: Date
}
```

### 3.4 SlotOverride
```swift
struct SlotOverride: Codable, Identifiable {
    let id: String
    let type: String                 // add | remove | block
    let date: String                 // YYYY-MM-DD
    let endDate: String?
    let startTime: String            // HH:MM
    let endTime: String              // HH:MM
    let label: String?
    let createdAt: Date
    let createdBy: String
}
```

---

## 4. Pending Implementation

### 4.1 Admin Sub-Views (Placeholder Views)
The following views are referenced in `AdminPortalView` but need implementation:
- `AdminDocumentsView`
- `AdminTimelineView`
- `AdminCourseOverviewView`
- `AdminRiadPortalView`
- `AdminFunnelManagerView`
- `AdminUser360View`
- `AdminInvitationsCouponsView`
- `AdminServiceOfferingsView`
- `AdminSessionOutcomesView`
- `AdminPriceEscalationView`
- `AdminProjectsView`
- `AdminUserManagementView`
- `AdminLinkManagerView`
- `AdminTapConfigView`

**Priority:** Medium (can be implemented incrementally per web portal parity)

### 4.2 Calendar Sync Integration
- **Status:** Cloud Function added, but not yet called in booking confirmation flow
- **Action:** Call `syncBookingToCalendar` after:
  - Free booking creation
  - Paid booking (`paymentStatus == "paid"`)
  - Manual confirmation (status → `confirmed`)

### 4.3 Verification Code Flow in Public Booking
- **Status:** Cloud Functions added, but UI not updated
- **Action:** Update `PublicSiteView.BookingLookupSection` to:
  1. Call `sendBookingVerifyCode(email)`
  2. Show code input field
  3. Call `verifyBookingCode(email, code)`
  4. If verified, call `findBookingsByEmail(email)`

### 4.4 PayPal Flow Integration
- **Status:** `PayPalPaymentView` created, but not integrated into booking flow
- **Action:** Add PayPal payment option in booking creation UI

---

## 5. Compilation Status

- **Last Attempt:** Build was canceled
- **Action Required:** Run `xcodebuild` to verify all changes compile
- **Potential Issues:**
  - New placeholder views in `AdminPortalView` may cause build errors
  - Missing view files need to be created or temporarily stubbed

---

## 6. Web Team Communication

### 6.1 Questions Sent (July 25)
- Documented in `IOS_REQUEST_TO_WEB_JULY25.md`
- Includes request for comprehensive web portal documentation (pages, routes, data, actions)

### 6.2 Responses Received (July 25)
- Documented in `WEB_REPLY_IOS_JULY25.md`
- All 8 questions answered
- Cloud Functions confirmed deployed
- Admin portal 4-category structure confirmed

### 6.3 Outstanding Questions
- **Q9 (Portal Documentation):** Awaiting web team response on comprehensive portal pages, routes, data, and actions

---

## 7. Roadmap

### Phase 1: Compilation & Stabilization (Immediate)
1. Run `xcodebuild` and fix any compile errors
2. Stub missing admin views to allow build to succeed
3. Test Cloud Function integrations

### Phase 2: Feature Integration (Short-term)
1. Integrate verification code flow in public booking
2. Integrate PayPal payment flow in booking creation
3. Add calendar sync calls after booking confirmation

### Phase 3: Admin Portal Parity (Medium-term)
1. Implement missing admin sub-views based on web portal
2. Align actions (CRUD + custom actions) with web
3. Add filtering, sorting, pagination to list views

### Phase 4: Web Portal Documentation (Long-term)
1. Receive comprehensive web portal documentation from web team
2. Audit iOS views against web pages
3. Remove unused iOS views
4. Ensure full parity between iOS and web admin portal

---

## 8. File Changes Summary

| File | Changes | Status |
|------|---------|--------|
| `Models.swift` | Updated Booking model fields | ✅ |
| `AdminModels.swift` | Updated BookingFull, Coupon; added SlotGroup, SlotOverride | ✅ |
| `FirestoreRepository.swift` | Added Cloud Functions, FirestoreTimestamp | ✅ |
| `AuthManager.swift` | Added findUserByEmail pre-check | ✅ |
| `PublicSiteView.swift` | Updated booking search to use CF | ✅ |
| `rsApp.swift` | Added PayPal deep link handler | ✅ |
| `PayPalPaymentView.swift` | New file for PayPal flow | ✅ |
| `AdminPortalView.swift` | Restructured to 4 categories | ✅ |
| `CalendarImportView.swift` | Updated Booking initializer | ✅ |

---

## 9. Dependencies

- Firebase SDK (Firestore, Auth, Functions)
- SafariServices (for PayPal SFSafariViewController)
- SwiftUI (iOS 15+)

---

## 10. Notes

- Custom URL scheme: `robotics-sciences://book/confirm?booking_id=...&paypal=1`
- Cloud Function region: `us-central1`
- Timestamp format from CFs: `{ _seconds, _nanoseconds }`
- Registration UX: Show sign-in prompt if `systemUsers` document exists

---

*Last updated: July 25, 2026*
