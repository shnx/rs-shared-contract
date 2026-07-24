# iOS → Web Team: Questions & Alignment — July 24

**From:** iOS team
**To:** Web team
**Date:** July 24, 2026
**Re:** Alignment after reviewing latest web docs, schema, Cloud Functions, and Firestore rules

---

## What We've Reviewed

We've read the following from the web repo (`/RS_website`):
- `docs/IOS_CHANGELOG_JULY24.md` — portal restructure, slot groups, coupon enhancements
- `docs/IOS_TEAM_GUIDE.md` — full system overview, tab structure, per-project visibility
- `docs/IOS_BOOKING_INTEGRATION.md` — discover flow, Tap payment, quotation system
- `src/types/index.ts` — all TypeScript interfaces (1,105 lines)
- `firestore.rules` — all security rules (249 lines)
- `functions/src/index.ts` — all Cloud Functions (4,127 lines)
- `src/components/BookingCenter.tsx` — public booking search with email verification
- `src/website/BookingConfirmPage.tsx` — confirmation page with animated guide + registration
- `src/website/DiscoverPage.tsx` — discover/booking flow with PayPal Hosted Buttons

---

## 1. Payment — PayPal is Active (Questions)

We see the DiscoverPage uses **PayPal Hosted Buttons** (`hostedButtonId: 'FTC4R6D22E42E'`) with the PayPal SDK loaded directly. But we also see Cloud Functions for **Tap** (`createTapCheckout`, `tapWebhook`), **Stripe** (`createStripeCheckout`, `stripeWebhook`), and **PayPal** (`createPayPalOrder`, `paypalIPN`).

**Questions:**
1. **Is PayPal the canonical payment method now?** The DiscoverPage loads PayPal SDK directly — not Tap or Stripe. Should iOS integrate PayPal only?
2. **PayPal Hosted Buttons vs PayPal API Orders** — The DiscoverPage uses `paypal.HostedButtons()` (rendered button), but there's also `createPayPalOrder` CF. Which flow should iOS use?
3. **Tap status** — Is Tap still active for any region/market? Should we keep the Tap admin config in iOS?
4. **Stripe status** — `createStripeCheckout` exists but Stripe secret key is commented out. Is Stripe deprecated?
5. **`resumeBookingPayment` CF** — This uses Stripe. If PayPal is canonical, should this be updated to use PayPal instead?
6. **Booking schema** — `Booking` has `stripeSessionId`, `tapChargeId`, `tapUrl`. Should we add `paypalOrderId` or `paypalTxnId`? The `paypalIPN` CF reuses `tapChargeId` for PayPal order ID — is this intentional?

---

## 2. Booking Confirmation Page — Already Has What We Need

The web `BookingConfirmPage.tsx` already has:
- ✅ Animated "What's next?" guide with 3 steps (fadeInUp animation)
- ✅ "Check your email" — confirmation sent
- ✅ "Find your booking anytime" — links to `/bookings`
- ✅ "Create an account" or "Log in" — detects if email exists via `findUserByEmail`
- ✅ Registration form (username + password) with username availability check
- ✅ Auto-suggests username from email
- ✅ Links booking to user account after registration

**This means iOS just needs to mirror this behavior.** We already have `BookingLookupSection` + `BookingRegistrationView` — we'll align the flow to match.

**Question:**
7. **Email exists detection** — `findUserByEmail` checks `systemUsers` collection. Is this reliable for both Firebase Auth users and legacy users? Should we also check `Auth.auth().fetchSignInMethods(forEmail:)`?

---

## 3. Public Booking Search — Email Verification

The web `BookingCenter.tsx` (public variant) has:
- Search by email or booking ID
- If email search: sends 6-digit verification code via `sendBookingVerifyCode` CF
- User must enter code to unlock management actions (cancel, etc.)
- Direct booking ID link = trusted (no verification needed)

**Our iOS `BookingLookupSection`** currently searches by name + email without verification. 

**Questions:**
8. **Should iOS add email verification?** The web requires it for destructive actions (cancel). Should we implement the same `sendBookingVerifyCode` flow?
9. **Name + email search vs email-only** — Web searches by email only (or booking ID). iOS searches by partial name + email. Should we align to email-only, or keep name+email as an extra filter?

---

## 4. Portal Restructure — 4 Categories

Web portal restructured from 6 to 4 categories:
- **Dashboard** — Documents, Analytics
- **People** — Funnel Manager, User Management, Communication, User Messages
- **Bookings & Services** — Calendar & Bookings, Invitations & Coupons, Service Offerings, Outcomes, Price Escalation, Proposals, Website Manager
- **Settings** — Projects, Payments, Admin Settings, Link Manager

**Our iOS `AdminPortalView`** currently has 10 sections in a flat list. We need to restructure to match.

**Questions:**
10. **Project Selector** — Web has project selection before tab bar. iOS currently doesn't. Should we add this? It's a significant UX change.
11. **Per-project tab visibility** — The matrix shows which tabs appear for which project type. Should iOS implement this filtering, or show all tabs for admin role?

---

## 5. New Collections & Schema Updates

### `slot_groups` (NEW)
We need to add this to iOS. Fields: `name`, `nameAr`, `slotTimes`, `maxSlotsPerDay`, `activeDays`, `isActive`, `priority`.

### `coupons` (UPDATED)
Added `slotGroup: String?` field. Also: `discountType` now includes `"fixed"` (was only `"free"` and `"percent"`).

### `meeting_invitations` (UPDATED)
Added `slotGroup: String?` field.

### `site_config/booking_settings` (NEW)
Dynamic booking config. iOS should read this instead of hardcoding.

**Questions:**
12. **`coupons.discountType` "fixed"** — The web type says `discountValue` is in cents for fixed type (1 = $0.01). Is this correct? The iOS `Coupon` model currently only handles "free" and "percent".
13. **`meeting_invitations` vs `meetingInvitations`** — Web type uses `meeting_invitations` (snake_case) but the CF and some code uses `meetingInvitations` (camelCase). Which is the actual Firestore collection name?
14. **`slot_overrides`** — New collection for one-off date overrides (add/remove/block). Should iOS implement this too?

---

## 6. CRM — Deprioritized

Per user direction, CRM is not a priority right now. We'll keep our existing `ady_crm` / `crm_contacts` dual-read but won't invest in new CRM features.

---

## 7. Cloud Functions — Full List Confirmed

We've confirmed all deployed CFs. New ones we need to integrate:
- `sendBookingVerifyCode` — for public booking email verification
- `createPayPalOrder` — for PayPal payment flow
- `resumeBookingPayment` — for resuming incomplete payments
- `resendBookingEmail` — for resending booking emails
- `sendBrandedPasswordReset` — branded password reset
- `sendBookingReminder` — scheduled reminder (admin trigger)
- `sendPasswordSetupEmail` — password setup email

**Question:**
15. **`syncBookingToCalendar`** — Confirmed deployed and working server-side. Should iOS use this CF exclusively (no client-side OAuth)? The doc says "no auth required, server-side service account."

---

## 8. Firestore Rules — Noted

Key rules that affect iOS:
- `bookings`: public `get` (by ID), public `create`, public `update` — **but `list` requires auth**. This means our `searchByNameAndEmail` (which does `querySafe(field: "email", isEqualTo:)`) will **fail for unauthenticated users**.
- `coupons`: public read, public update (for usage increment)
- `slot_groups`: public read
- `service_offerings`: public read
- `availability`: public read
- `site_config`: public read

**Critical Question:**
16. **Booking search for unauthenticated users** — Firestore rules say `allow list: if request.auth != null`. Our `BookingsRepository.searchByNameAndEmail()` uses a Firestore query (`whereField("email", isEqualTo:)`) which is a `list` operation. **This will fail for unauthenticated users.** How does the web handle this? The web `BookingCenter` calls `bookingDB.getByEmail()` — does this work because the portal uses JWT auth (not Firebase Auth)? For iOS public users (not logged in), we need either:
    - a Cloud Function to search bookings by email, OR
    - a Firestore rule change to allow public list by email, OR
    - require authentication before searching

---

## 9. Timeline — Confirmed Renamed

Web uses `timeline` collection. We've already updated `TimelineRepository` to use `timeline` as primary with `timeline_entries` fallback. ✅

---

## 10. Availability — Confirmed Recurring Weekly

Web canonical schema is `dayOfWeek` + `startTime`/`endTime` strings. iOS already aligned. ✅

---

## Summary of What We Need From Web Team

| # | Question | Priority |
|---|----------|----------|
| 1 | Is PayPal the canonical payment method? | HIGH |
| 2 | PayPal Hosted Buttons vs PayPal API Orders for iOS? | HIGH |
| 3 | Should we keep Tap admin config in iOS? | MEDIUM |
| 4 | Is Stripe deprecated? | LOW |
| 5 | Should `resumeBookingPayment` use PayPal? | MEDIUM |
| 6 | Add `paypalOrderId` to Booking schema? | MEDIUM |
| 7 | Email exists detection method? | LOW |
| 8 | Should iOS add email verification for booking search? | MEDIUM |
| 9 | Align to email-only search or keep name+email? | LOW |
| 10 | Should iOS add project selector? | MEDIUM |
| 11 | Per-project tab visibility for iOS? | MEDIUM |
| 12 | Coupon "fixed" discountType in cents? | LOW |
| 13 | `meeting_invitations` vs `meetingInvitations` collection name? | MEDIUM |
| 14 | Should iOS implement `slot_overrides`? | LOW |
| 15 | Use `syncBookingToCalendar` CF exclusively? | LOW |
| 16 | **How to search bookings for unauthenticated users?** | **CRITICAL** |

---

## What iOS Will Do Next (Regardless of Answers)

1. ✅ Add `SlotGroup` model + repository
2. ✅ Add `BookingCalendarConfig` model (read from `site_config/booking_settings`)
3. ✅ Update `Coupon` model with `slotGroup` and `fixed` discount type
4. ✅ Restructure `AdminPortalView` to 4-category tab structure
5. ✅ Add slot groups management UI
6. ✅ Update coupon management with toggle, edit, reset, status badges
7. ✅ Integrate `sendBookingVerifyCode` CF for booking search verification
8. ✅ Add PayPal payment flow (pending confirmation on approach)
9. ✅ Mirror BookingConfirmPage animated guide on iOS

*Shared in `shared/docs/IOS_QUESTIONS_JULY24.md`*
