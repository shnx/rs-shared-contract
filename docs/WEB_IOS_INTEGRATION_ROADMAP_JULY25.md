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
  ├─ Tap    → createTapCheckout    → open tapUrl
  ├─ Stripe → createStripeCheckout → open stripeUrl
  └─ PayPal → createPayPalOrder    → open url in SFSafariViewController
  │
  Webhook/ipn marks booking paymentStatus = paid
  │
  Poll bookings/{id} until paid
  │
  syncBookingToCalendar(bookingId)
```

## 3. Cloud Function cheat sheet

| Use | CF | Auth |
|---|---|---|
| Public email search | `findBookingsByEmail` | None |
| Send verify code | `sendBookingVerifyCode` | None |
| Verify code | `verifyBookingCode` | None |
| Tap pay | `createTapCheckout` | None |
| Stripe pay | `createStripeCheckout` | None |
| PayPal pay | `createPayPalOrder` | None |
| Calendar event | `syncBookingToCalendar` | None |

## 4. iOS implementation order

1. Public search with `findBookingsByEmail` (use JSONDecoder, `_seconds` timestamps).
2. Verification with `sendBookingVerifyCode` + `verifyBookingCode`.
3. Payment flows: Tap → Stripe → PayPal.
4. `registerFromBooking` and Sign In handling.
5. Admin portal 4 categories + `user-360`.

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

*Use `WEB_REPLY_IOS_JULY25.md` for exact payloads and `WEB_REPLY_IOS_JULY24.md` for the original Q&A.*
