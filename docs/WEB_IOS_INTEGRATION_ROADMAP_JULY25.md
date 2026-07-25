# Web → iOS Integration Roadmap — July 25, 2026

Companion to `WEB_REPLY_IOS_JULY25.md`. Big-picture implementation guide.

---

## 1. Public booking flow

```
Enter email → sendBookingVerifyCode(email)
              → User gets 6-digit code
              → verifyBookingCode(email, code)
              → findBookingsByEmail(email)
              → Show booking list
              → Optional: registerFromBooking(email, password)
```

If `systemUsers` already exists → show **Sign In** prompt, do not merge.

## 2. Paid booking flow

```
Select slot + coupon
  │
  ├─ PayPal (primary) → createPayPalOrder → open url in SFSafariViewController
  │   Resume unpaid   → resumeBookingPayment → open url
  ├─ Tap (legacy)     → createTapCheckout    → open tapUrl
  └─ Stripe (legacy)  → createStripeCheckout → open stripeUrl
  │
  Webhook/ipn marks booking paymentStatus = paid
  │
  Poll bookings/{id} until paid
  │
  syncBookingToCalendar(bookingId)
  │
  NOTE: No auto confirmation emails — disabled per admin request
```

## 3. Cloud Function cheat sheet

| Use | CF | Auth |
|---|---|---|
| Public email search | `findBookingsByEmail` | None |
| Send verify code | `sendBookingVerifyCode` | None |
| Verify code | `verifyBookingCode` | None |
| **PayPal pay (primary)** | `createPayPalOrder` | None |
| **Resume unpaid PayPal** | `resumeBookingPayment` | None |
| Tap pay (legacy) | `createTapCheckout` | None |
| Stripe pay (legacy) | `createStripeCheckout` | None |
| Calendar event | `syncBookingToCalendar` | None |
| **Password reset** | `sendBrandedPasswordReset` | Admin |

## 4. iOS implementation order

1. Public search with `findBookingsByEmail` (use JSONDecoder, `_seconds` timestamps).
2. Verification with `sendBookingVerifyCode` + `verifyBookingCode`.
3. Payment flows: **PayPal first** (`createPayPalOrder` + `resumeBookingPayment`), then Tap/Stripe as fallback.
4. `registerFromBooking` and Sign In handling.
5. **Student Portal** for `candidate`/`student`/`graduate` roles (restricted — no admin tabs).
6. Admin portal 4 categories + `user-360`.
7. User Management: add bookings info per user + password reset action.

## 5. Admin portal 4 categories

```
Dashboard
  documents, timeline (accounting), course-overview (education), riad-portal (real_estate)
People
  funnel-manager, user-360
Bookings & Services
  calendar, invitations-coupons, service-offerings, session-outcomes, price-escalation
Settings
  projects, user-management, admin-settings, link-manager, tap-config
```

## 6. Important schema notes

- `paymentMethod` is required: `"tap" | "stripe" | "paypal" | "manual" | "free"`.
- Use `paypalOrderId` and `paypalTransactionId` for PayPal; do not use `tapChargeId` for PayPal.
- Timestamps from CFs are `{ _seconds, _nanoseconds }` maps; use `JSONDecoder`.
- `slot_groups.slotGroup` and `coupons.slotGroup` are IDs referencing `slot_groups`.
- `slot_overrides` apply globally to calendar generation.

---

*Use `WEB_REPLY_IOS_JULY25.md` for exact payloads and `WEB_REPLY_IOS_JULY24.md` for the original Q&A. See `IOS_UPDATE_JULY25_PORTAL_CHANGES.md` for the latest portal changes (student portal, PayPal primary, user management, password reset).*
