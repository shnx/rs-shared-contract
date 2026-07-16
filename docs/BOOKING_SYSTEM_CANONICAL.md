# Booking & Services — Canonical Spec (Source of Truth)

**Date:** July 8, 2026
**Status:** Approved — implement identically on web + iOS
**Supersedes:** the pricing sections of `SERVICES_BOOKING_PROPOSAL.md` and
`IOS_PROPOSAL_PAID_CONSULTATIONS.md`. Those remain useful for the surrounding
funnel/quotation ideas, but **this file is the authoritative pricing and data
model** both apps must match.

Pricing decision (July 8): **USD funnel tiers** (option B). All prices in USD.
Show `~~originalPrice~~ currentPrice` wherever a discount exists.

---

## Service Catalog (`service_offerings`)

Seed these four offerings. `id` == `type` so both apps can reference a stable slug.

| id / type | name | audience | billing | priceUSD | originalPriceUSD | duration | requiresScheduling |
|-----------|------|----------|---------|----------|------------------|----------|--------------------|
| `coffee_time` | Coffee Time | student | one_time | 7 | 15 | 30 min | yes |
| `curriculum_builder` | Curriculum Builder | student | monthly | 25 | 40 | 60 min | no (subscription) |
| `course_access` | Course Access | student | monthly | 12 | 20 | 0 | no (subscription) |
| `company_assessment` | System Assessment | company | hourly | 50 | — | 60 min | yes |

- `company_assessment` is ranged: `priceUSD = 50`, `priceMaxUSD = 100` (per hour).
- Membership tiers (`curriculum_builder`, `course_access`) do **not** book a
  time slot; they create a `pending`/`unpaid` booking that represents the
  subscription intent until payment is wired up.

Full field list: `schemas/service-offering.json`.

---

## Collections

| Collection | Writer | Reader | Schema |
|------------|--------|--------|--------|
| `service_offerings` | web admin | web + iOS | `schemas/service-offering.json` |
| `bookings` | web + iOS (public create) | web admin + iOS | `schemas/booking.json` |
| `availability` | web admin | web + iOS | `schemas/availability.json` |

Names are **unprefixed** (platform-wide, not ADY-specific) — consistent with the
`crm_contacts` / `job_submissions` convention already in use.

### Slot generation
`availability` stores recurring weekly windows (`dayOfWeek`, `startTime`,
`endTime`, `slotMinutes` default 30, `bufferMinutes` default 15,
`maxBookingsPerDay` default 5, `timezone` default `Asia/Amman`). The client
expands the next N days of windows into concrete slots and hides any slot that
already has a `confirmed`/`pending` booking at that `scheduledAt`.

If no `availability` docs exist, both apps fall back to a **default schedule**:
Sun–Thu, 10:00–18:00, 30-min slots, 15-min buffer.

---

## Booking flow (both apps)

1. Visitor picks a service offering.
2. If `requiresScheduling`, pick a slot from the availability calendar.
3. Enter name, email, (company for company_assessment), notes.
4. Create a `bookings` doc: `status='pending'`, `paymentStatus='unpaid'`,
   `priceUSD` copied from the offering, `serviceType` denormalized.
5. Payment: Stripe Checkout (`createStripeCheckout` → `stripeWebhook` marks
   `paid` + `confirmed`). Until Stripe keys are provisioned, the booking stays
   `pending`/`unpaid` and the admin confirms manually from the portal.

---

## Consistency rules

- `serviceType` and `status`/`paymentStatus` enums must match exactly across
  web (`src/types`) and iOS (`Models.swift`).
- Money is always USD major units (a number, e.g. `7`).
- Dates use the shared date-tolerant convention (Timestamp | ISO string | epoch
  | `{seconds,nanoseconds}`).
- Any change here → bump `CHANGELOG.md` and update **both** apps.
