# iOS Update — Portal Changes, Student Portal & Payment — July 25, 2026

This document covers all web changes the iOS team must mirror. Read this carefully — several new features and restrictions were added today.

---

## 1. No localhost anywhere — production URLs only

All URLs in the codebase use `https://the-rs.com` as the base. No localhost references exist in:

- **Cloud Functions** (`functions/src/index.ts`): `SITE_URL = process.env.SITE_URL || "https://the-rs.com"`
- **Frontend** (`src/utils/emailConfig.ts`): Falls back to `https://the-rs.com` when `window.location.origin` includes localhost.
- **PayPal return URLs**: `${SITE_URL}/book/confirm?booking_id={id}&paypal=1`
- **Stripe success/cancel URLs**: `${SITE_URL}/book/confirm?booking_id={id}&stripe=1` / `${SITE_URL}/discover?cancelled=1`
- **Tap redirect URLs**: `${SITE_URL}/book/confirm?booking_id={id}`
- **Password reset URLs**: `${SITE_URL}/reset-password?mode=resetPassword&oobCode={code}&lang=en`
- **Registration links**: `${SITE_URL}/register?invite={id}`

**iOS action:** Ensure all deep links and callback URLs use `https://the-rs.com`, not localhost or any dev URL.

---

## 2. Student/Candidate Portal — Restricted Access

### Who gets the Student Portal

The following roles now ALL route to the **Student Portal** (not the admin portal):

| Role | Portal | Access |
|---|---|---|
| `candidate` | StudentPortal | Sessions, Book, CV, Ask Admin, Community, Inbox, Settings |
| `student` | StudentPortal | Same as candidate |
| `graduate` | StudentPortal | Same as candidate |
| `client` | ClientPortal | Overview, Sessions, Documents, Messages, Settings |
| `admin` | Admin Portal | Full access |
| `manager` | Admin Portal | Full access (minus admin-only tools) |
| `advisor` | Admin Portal | Limited tabs |

**iOS action:** iOS must route `candidate`, `student`, and `graduate` roles to a Student Portal screen — NOT the admin portal. No analytics, no user management, no funnel manager, no finance tabs.

### Student Portal tabs

The Student Portal shows only these tabs (in order):

1. **Submission** — shows the user's job submission status (if any)
2. **Sessions** — shows the user's bookings with status, payment status, price, and payment method
3. **Book Session** — in-portal booking page with PayPal payment (same as public Discover page)
4. **My CV** — shows the user's CV file (if submitted)
5. **Ask Admin** — send a message to admin team
6. **Community** — candidate community feed
7. **Inbox** — admin replies and notifications
8. **Settings** — change username, password, account preferences

### Data loading — user's own data only

The Student Portal loads ONLY the current user's data:

```
bookings:  bookingDB.getByEmail(currentUser.email)   // only their bookings
submission: submissions.filter(s => s.email == currentUser.email)  // only their submission
contact:   crmContacts.find(c => c.email == currentUser.email)     // only their CRM record
```

**iOS action:** Never load all bookings/submissions/contacts for a student. Always filter by `currentUser.email`.

### PayPal payment from Student Portal

Unpaid bookings show a **"Pay with PayPal"** button that calls:

```
resumeBookingPayment(bookingId) → { success, url }
```

This CF checks if the booking is already paid. If not, it creates a new PayPal order and returns the approval URL. The user is redirected to PayPal, then back to `/book/confirm?booking_id={id}&paypal=1`.

**iOS action:** Implement `resumeBookingPayment` CF call for unpaid bookings in the student portal.

---

## 3. Admin: Booking Preview Tab

A new admin tab **"Booking Preview"** (id: `booking-preview`) has been added under the **Bookings** category. It renders the public `DiscoverPage` component inside the admin portal so admins can see exactly what public users see when booking.

- **Roles:** `admin`, `manager`
- **Project types:** `website`
- **Category:** `bookings`
- **Features:** Desktop/mobile toggle, "Open Full Page" link to `/discover`

**iOS action:** Optional — add a preview screen in the admin app that shows the public booking page. Not critical but nice for admin QA.

---

## 4. Admin: User Management Improvements

### New columns in user table

The User Management table now shows:

| Column | Data source | Description |
|---|---|---|
| **Bookings** | `bookings` collection filtered by user email | Total count, paid count, unpaid count, last booking date |
| **Submission** | `job_submissions` filtered by user email | Submission status badge (new, reviewing, shortlisted, invited, enrolled, declined, archived) |

### New stat card

- **"With Bookings"** — count of users who have at least one booking

### New action button: Reset Password

Each user row now has a **"Reset PW"** button (amber, key icon) that calls:

```
sendBrandedPasswordReset(email) → { success, message }
```

This CF generates a Firebase password reset link and sends a branded email with a reset URL:
```
https://the-rs.com/reset-password?mode=resetPassword&oobCode={code}&lang=en
```

**iOS action:** Add a "Reset Password" action to user rows in the admin app. Call `sendBrandedPasswordReset` CF.

---

## 5. Payment Flow — PayPal (Primary)

### PayPal is the primary payment method

The booking flow now uses **PayPal** as the primary payment method across:

- **Public Discover page** (`/discover`) — `createPayPalOrder`
- **Meeting Booking page** (`/book`) — `createPayPalOrder`
- **Student Portal** (in-portal booking) — `createPayPalOrder`
- **Student Portal** (resume unpaid booking) — `resumeBookingPayment`

### createPayPalOrder CF

**Input:**
```json
{
  "bookingId": "booking_doc_id",
  "email": "user@example.com",
  "name": "User Name",
  "scheduledAt": "2026-08-01T10:00:00.000Z",
  "notes": "Consultation",
  "finalPrice": 4.5
}
```

**Output:**
```json
{
  "success": true,
  "url": "https://www.paypal.com/checkoutnow?token=ORDER_ID"
}
```

**Flow:**
1. CF fetches PayPal credentials from `paypal_config` Firestore collection
2. Gets PayPal access token via `https://api.paypal.com/v1/oauth2/token`
3. Creates PayPal order via `https://api.paypal.com/v2/checkout/orders`
4. Sets `return_url` = `${SITE_URL}/book/confirm?booking_id={id}&paypal=1`
5. Sets `cancel_url` = `${SITE_URL}/discover?cancelled=1`
6. Updates booking with `paypalOrderId`
7. Returns PayPal approval URL

### resumeBookingPayment CF

**Input:**
```json
{ "bookingId": "booking_doc_id" }
```

**Output:**
```json
{
  "success": true,
  "url": "https://www.paypal.com/checkoutnow?token=ORDER_ID"
}
```

If booking is already paid, returns `{ success: true, url: "${SITE_URL}/book/confirm?booking_id={id}" }`.

### PayPal credentials storage

PayPal credentials are stored in Firestore `paypal_config` collection:

```
paypal_config/{configId}:
{
  clientId: string,
  clientSecret: string,
  mode: "live" | "sandbox",
  active: boolean
}
```

**iOS action:** Use `createPayPalOrder` for all payment flows. Use `resumeBookingPayment` for unpaid bookings in student portal. Open the PayPal URL in `SFSafariViewController` and handle the redirect back to the app.

---

## 6. Booking Confirmation Page

URL: `/book/confirm?booking_id={id}&paypal=1`

### What it does

1. Reads `booking_id` from URL query params or `localStorage.pending_booking_id`
2. Fetches booking document from Firestore
3. If `paypal=1` in query params → polls booking for payment status update
4. When `paymentStatus === 'paid'`:
   - Updates booking `status` to `confirmed`
   - Syncs to Google Calendar via `syncBookingToCalendar`
   - Shows success screen with booking details
5. If still `pending` after 3 seconds → shows "Payment processing" screen

### Auto emails DISABLED

The booking confirmation page **no longer sends automatic confirmation emails**. The `sendBookingConfirmationEmail` call has been commented out per admin request.

**iOS action:** Do not auto-send confirmation emails from the booking confirmation flow. Admins can still send emails manually from the Communication tab or Booking Settings.

---

## 7. Booking schema — full field reference

```
bookings/{id}:
{
  email: string
  name: string
  scheduledAt: Timestamp
  durationMinutes: number
  status: "pending" | "confirmed" | "completed" | "cancelled"
  paymentStatus: "paid" | "unpaid"
  paymentMethod: "paypal" | "tap" | "stripe" | "manual" | "free"
  priceUSD: number
  serviceOfferingId: string
  notes: string
  couponCode: string (optional)
  paypalOrderId: string (optional)
  paypalTransactionId: string (optional)
  tapChargeId: string (optional)
  tapUrl: string (optional)
  stripeSessionId: string (optional)
  sessionLink: string (optional)  // e.g. "https://meet.jit.si/robotics-sciences-{id}"
  userId: string (optional)       // linked Firebase Auth UID
  createdAt: Timestamp
  updatedAt: Timestamp
}
```

---

## 8. Cloud Function reference (updated)

| CF | Purpose | Auth | New? |
|---|---|---|---|
| `createPayPalOrder` | Create PayPal order for booking | None | No (existing, now primary) |
| `resumeBookingPayment` | Resume payment for unpaid booking | None | No (existing, now used in student portal) |
| `createTapCheckout` | Create Tap checkout | None | No (legacy) |
| `createStripeCheckout` | Create Stripe checkout session | None | No (legacy) |
| `sendBrandedPasswordReset` | Send branded password reset email | Admin | No (existing, now exposed in user management) |
| `sendBookingConfirmationEmail` | Send booking confirmation email | None | **Disabled in auto flow** — still callable manually |
| `syncBookingToCalendar` | Sync booking to Google Calendar | None | No |
| `findBookingsByEmail` | Find bookings by email (public) | None | No |
| `sendBookingVerifyCode` | Send 6-digit verification code | None | No |
| `verifyBookingCode` | Verify 6-digit code | None | No |
| `deleteUserData` | Delete all user data + Firebase Auth | Admin | No |

---

## 9. Firestore collections reference

| Collection | Purpose |
|---|---|
| `bookings` | All booking records |
| `job_submissions` | Candidate applications |
| `systemUsers` | User profiles (role, email, projectIds, permissions) |
| `student_crm` | Student CRM contacts |
| `paypal_config` | PayPal credentials (clientId, clientSecret, mode, active) |
| `tap_config` | Tap gateway credentials (legacy) |
| `service_offerings` | Bookable services (consultation, mentorship, MVP) |
| `slot_overrides` | Calendar slot overrides |
| `calendar_blocks` | Blocked calendar dates |
| `coupons` | Discount coupons |
| `session_outcomes` | Post-session proof-of-work |
| `user_messages` | Student-to-admin messages |
| `portal_messages` | Admin-to-student messages |
| `email_logs` | Email send history |
| `analytics_sessions` | Visitor analytics |
| `analytics_events` | Visitor events |

---

## 10. iOS implementation priority

1. **Student Portal** — route `candidate`/`student`/`graduate` to a restricted portal with only their data
2. **PayPal payment** — use `createPayPalOrder` as primary, `resumeBookingPayment` for unpaid bookings
3. **Password reset** — add "Reset Password" action in admin user management calling `sendBrandedPasswordReset`
4. **No auto emails** — do not send booking confirmation emails automatically
5. **Production URLs only** — all URLs must use `https://the-rs.com`

---

*Questions? Check `IOS_PORTAL_ALIGNMENT_JULY25.md` for the full admin portal tab mapping and `WEB_IOS_INTEGRATION_ROADMAP_JULY25.md` for the public booking flow.*
