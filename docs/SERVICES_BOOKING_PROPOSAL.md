# Services & Booking Funnel — Proposal for iOS Team

**Date:** July 7, 2026
**From:** Web team
**Status:** Proposal — seeking iOS feedback

---

## Summary

We're transforming the website from a passive CV-submission form into a service-offering platform. Two audiences:

- **Students/Individuals**: "Coffee Time" — $7 (was $15) casual 30-min consultation, with upsell to membership plans
- **Companies**: "System Assessment" — $50–100/hour consultation booking

Payments via Stripe. Custom booking calendar. All prices in USD.

---

## New Firestore Collections

### `service_offerings` (admin-managed service catalog)
| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Auto-generated |
| `name` | string | e.g. "Coffee Time" |
| `nameAr` | string? | Arabic name |
| `description` | string | |
| `descriptionAr` | string? | |
| `type` | `'coffee_time' \| 'company_assessment' \| 'membership' \| 'course_access'` | |
| `audience` | `'student' \| 'company'` | |
| `priceUSD` | number | Current price |
| `originalPriceUSD` | number? | Original price (for discount display) |
| `durationMinutes` | number | |
| `isActive` | boolean | |
| `createdAt` | timestamp | |

### `bookings` (consultation bookings)
| Field | Type | Description |
|-------|------|-------------|
| `id` | string | |
| `userId` | string? | If logged in |
| `email` | string | |
| `name` | string | |
| `company` | string? | |
| `serviceOfferingId` | string | |
| `serviceType` | string | Denormalized from offering |
| `scheduledAt` | timestamp | |
| `durationMinutes` | number | |
| `status` | `'pending' \| 'confirmed' \| 'completed' \| 'cancelled'` | |
| `priceUSD` | number | |
| `stripeSessionId` | string? | |
| `paymentStatus` | `'unpaid' \| 'paid'` | |
| `notes` | string? | |
| `createdAt` | timestamp | |

### `availability` (admin's available time slots)
| Field | Type | Description |
|-------|------|-------------|
| `id` | string | |
| `dayOfWeek` | number (0–6) | 0 = Sunday |
| `startTime` | string | e.g. "10:00" |
| `endTime` | string | e.g. "18:00" |
| `bufferMinutes` | number | Default 15 |
| `maxBookingsPerDay` | number | Default 5 |

---

## New Cloud Functions

| Function | Type | Purpose |
|----------|------|---------|
| `createStripeCheckout` | callable | Creates Stripe Checkout Session, stores pending booking |
| `stripeWebhook` | HTTP | Receives Stripe webhook, marks booking paid + confirmed, sends email |
| `sendOfferEmail` | callable | Sends branded offer email to student with 3 upsell options |

### `sendOfferEmail` Details
- Input: `{ jobSubmissionId, offerType }`
- `offerType`: `'coffee_time' \| 'membership' \| 'course_access' \| 'full_funnel'`
- Sends branded email with 3 option cards:
  1. **Coffee Time** — ~~$15~~ $7 — single 30-min session
  2. **Curriculum Builder** — ~~$40/mo~~ $25/mo — 4 sessions/month + custom curriculum
  3. **Course Access** — ~~$20/mo~~ $12/mo — all recorded courses + early access
- Each card has a booking link: `/book?offer={token}&type={type}`
- Updates `job_submissions` with `offerSentAt` timestamp

---

## New Public Pages

### `/book` — Booking Page
- Two paths: Student ("Coffee Time" $7) and Company ("System Assessment" $50+)
- Custom calendar picker from `availability` collection
- Booking form → Stripe Checkout → confirmation email

### Updated Pages
- **HomePage**: Add services CTA section
- **ServicesPage**: Add "Book a Consultation" CTA with pricing
- **JoinUs**: After CV submission, show "Book a Coffee Time → $7" CTA
- **Navigation**: Add "Book" nav item

---

## Membership Integration

Repurpose existing `MembershipPlan` system with 3 default tiers:

| Plan | Price | Billing | Includes |
|------|-------|---------|----------|
| Coffee Time | $7 | one-time | Single 30-min session |
| Curriculum Builder | $25/mo (was $40) | monthly | 4 sessions/month + custom curriculum |
| Course Access | $12/mo (was $20) | monthly | All recorded courses + early access |

---

## Questions for iOS Team

1. **Do you want to show bookings in the iOS app?** We'll expose `bookings` and `availability` collections. iOS could show upcoming appointments, booking history, etc.

2. **Should iOS also support booking?** The web handles booking + payment, but iOS could display the same service offerings and redirect to web for checkout, or implement native Stripe SDK.

3. **`service_offerings` naming**: We're using snake_case per our convention. iOS should read this collection for displaying service prices/descriptions if needed.

4. **Offer email flow**: When admin sends an offer email to a student, should iOS also show a notification or banner in the app? We can add a field to `job_submissions` like `offerSentAt` and `offerType` for iOS to read.

5. **Availability sync**: If iOS also manages availability, we need to agree on who writes to `availability`. Proposal: web admin manages it, iOS reads it.

6. **Stripe**: Web handles Stripe Checkout (hosted page). iOS would need Stripe SDK for native payments. Is that something you want to build, or should iOS redirect to the web booking page?

7. **Currency**: All prices in USD. Does iOS need to show JOD equivalent? We can add a conversion field if needed.

---

## Implementation Timeline

| Phase | What | When |
|-------|------|------|
| 1 | Data model + Stripe Cloud Functions | This week |
| 2 | Booking page (`/book`) | This week |
| 3 | Offer email automation | Next week |
| 4 | Admin portal (bookings, availability, offerings) | Next week |
| 5 | Membership integration + seed data | Next week |
| 6 | Update existing pages + test end-to-end | Next week |

---

## Technical Notes

- **Stripe**: Checkout (hosted page). 2.9% + $0.30/transaction. No PCI compliance needed.
- **Calendar**: Custom slot picker from `availability` collection. No external dependency.
- **Email**: Reuses existing Resend SMTP infrastructure with anti-spam headers.
- **Bilingual**: All new pages support EN/AR.
- **Discount framing**: Always show `~~originalPrice~~` next to current price.
