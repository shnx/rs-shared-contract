# iOS → Web Team Request – July 25, 2026

This document captures the remaining open questions after applying the July 24 web team answers to the iOS codebase.

---

## P1 fixes already applied

- `Booking` model aligned to the canonical schema:
  - Removed: `consultationId`, `tier`, `priceEUR`, `invoiceUrl`
  - Added: `couponCode`, `tapChargeId`, `tapUrl`, `paypalOrderId`, `paypalTransactionId`, `sessionLink`, `paymentMethod` (tap/stripe/paypal)
- `BookingFull` and `Coupon` updated similarly (`slotGroup`, `discountType` = free/percent/fixed).
- Public booking lookup now calls the `findBookingsByEmail` Cloud Function and no longer filters client-side by name.
- `BookingsRepository` now exposes `findBookingsByEmail` and `sendBookingVerifyCode`.
- `AuthManager` now uses `findUserByEmail` before attempting `registerFromBooking`.

---

## Remaining questions for the TENA / web team

### 1. `paymentMethod` field ambiguity

In Part 1 Q4 the canonical `Booking` schema says to **drop** `paymentMethod` (old values: `stripe` / `manual` / `invoice`).  
In Part 6 Q6 it says to **add** `paymentMethod` (values: `tap` / `stripe` / `paypal`).

**Request:** Confirm whether `Booking` should keep `paymentMethod` with the new gateway semantics, or drop it entirely. iOS currently keeps it with the new semantics.

---

### 2. `findBookingsByEmail` response shape

**Request:** Confirm the exact JSON / serialized shape returned by `findBookingsByEmail`, especially:

- Are timestamps returned as Firestore `Timestamp` objects, ISO strings, or `{ _seconds, _nanoseconds }` maps?
- Is each booking dictionary keyed by `id` or by `documentId`?

iOS currently decodes with `Firestore.Decoder`; if the CF returns plain JSON this may need to be switched to `JSONDecoder`.

---

### 3. `sendBookingVerifyCode` verification endpoint

`sendBookingVerifyCode` sends a 6-digit code, but iOS also needs to **verify** that code before a destructive action.

**Request:** Provide the verification Cloud Function name and expected payload/response (e.g., `verifyBookingCode` with `{ email, bookingId, code }`).

---

### 4. Registration flow when `systemUsers` already exists

`registerFromBooking` now checks `systemUsers` by email and throws `emailAlreadyInUse` if a document exists.

**Request:** What should the iOS UX be when a `systemUsers` document exists but the user has never set a Firebase password? Should iOS:
- Show a "Sign in" prompt, or
- Allow the user to set a password and link the Firebase account?

---

### 5. `createPayPalOrder` iOS integration details

**Request:** Provide:

- Exact request payload for `createPayPalOrder` (which `bookingId`, `returnUrl`, `cancelUrl`).
- Exact response shape and how iOS should extract `paypalOrderId` and `paypalTransactionId`.
- The PayPal redirect URLs iOS should register as custom URL schemes or capture via `SFSafariViewController`.

---

### 6. When to call `syncBookingToCalendar` and `sendBookingConfirmation`

**Request:** Confirm:

- Are these called automatically by the backend after `createPayPalOrder` succeeds, or must iOS call them explicitly?
- If iOS must call them, what are the payloads and when (after free booking, after paid booking, after status change)?

---

### 7. `slot_groups` and `slot_overrides` schemas

**Request:** Provide the exact collection names, field lists, and sample documents for:

- `slot_groups`
- `slot_overrides`
- How they link to `serviceOfferings` and `bookings`

---

### 8. Admin portal 4-category restructure spec

**Request:** Provide the exact iOS admin portal categories, tab names, and project-selector behavior, or a link to the authoritative spec (e.g., `IOS_CHANGELOG_JULY24.md` sections 1-8). iOS will restructure `AdminPortalView` once the spec is confirmed.

---

## Action requested from web team

Please reply to the above questions in the same format as `WEB_REPLY_IOS_JULY24.md` so iOS can continue the remaining P2/P3 fixes in the next iteration.
