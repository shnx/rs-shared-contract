# iOS Response to Services & Booking Proposal

**From:** iOS team
**To:** Web team
**Date:** July 8, 2026
**Re:** Reconciling SERVICES_BOOKING_PROPOSAL with MARKETING_STRATEGY_ECOSYSTEM

---

## Summary

We've just completed a marketing strategy document based on consultation with ChatGPT about positioning, pricing, and ecosystem approach. There are conflicts between your proposal and the marketing strategy that need reconciliation.

**Core conflict:** Your proposal is transactional (funnel with upsells), while the marketing strategy is relationship-first (ecosystem with long-term nurturing).

---

## Key Conflicts & Recommendations

### 1. Entry Point: $7 Coffee Time vs €15 First Coffee

**Web Team:** $7 Coffee Time (30 min)
**Marketing Strategy:** €15 First Coffee (30 min)

**Recommendation:** Use €15 First Coffee.

**Reasoning:**
- €15 is intentionally positioned as "price of a coffee for both of us" — it's a story, not a discount
- $7 feels like a discount tactic (~~$15~~ $7) which hurts brand perception
- The marketing strategy emphasizes that the low price is intentional to remove barriers for passionate students, not to upsell them
- €15 is in EUR because the target audience (students in Germany/Europe) relates to EUR

### 2. Currency: USD vs EUR

**Web Team:** USD
**Marketing Strategy:** EUR

**Recommendation:** Use EUR for student-facing services, USD for company-facing services.

**Reasoning:**
- Students (Europe/Germany market) think in EUR
- Companies (international) often operate in USD
- We can store both `priceEUR` and `priceUSD` fields and display based on audience

### 3. Philosophy: Funnel vs Ecosystem

**Web Team:** Funnel — Coffee Time → upsell to membership/course access
**Marketing Strategy:** Ecosystem — First Coffee → assessment → nurture → mentoring → community

**Recommendation:** Hybrid approach.

**How it works:**
- **Entry:** First Coffee (€15) — no upsell, just relationship starter
- **Assessment:** After First Coffee, send thoughtful questions (excitement, blockers, 1-year vision)
- **Nurture:** Monthly community digest (papers, tools, internships, challenges) — free value
- **Conversion:** When ready, offer mentoring (€60-120) or company advisory (€350-600)
- **Membership:** Only offer membership after they've engaged with content/community

**Why:** The funnel approach feels like "get them in, extract value." The ecosystem approach feels like "build community, value emerges naturally."

### 4. Collection Naming: service_offerings vs consultations

**Web Team:** `service_offerings`
**Marketing Strategy:** `consultations`

**Recommendation:** Use `consultations` for the service catalog.

**Reasoning:**
- "Consultations" is clearer and more professional
- Aligns with the consultation/booking system we proposed in `IOS_PROPOSAL_PAID_CONSULTATIONS.md`
- "Service offerings" is generic — consultations is specific to what we're doing

### 5. Payment: Stripe vs Manual

**Web Team:** Stripe Checkout
**Marketing Strategy:** Not specified

**Recommendation:** Stripe is good, but add manual option for companies.

**Reasoning:**
- Students: Stripe (low friction, instant)
- Companies: Manual (invoices, bank transfer, corporate procurement)
- Companies often can't pay via Stripe Checkout — they need invoices

---

## Unified Collection Schema

### `consultations` (service catalog)

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | doc id |
| `title` | string | |
| `titleAr` | string? | Arabic |
| `description` | string | |
| `descriptionAr` | string? | Arabic |
| `tier` | string | `first_coffee`, `mentoring`, `professional`, `company`, `workshop`, `advisor` |
| `category` | string | `training`, `advisory`, `workshop`, `technical`, `career` |
| `audience` | string | `student`, `professional`, `company` |
| `durationMinutes` | number | |
| `priceEUR` | number? | EUR price |
| `priceUSD` | number? | USD price |
| `originalPriceEUR` | number? | For discount display (if needed) |
| `originalPriceUSD` | number? | For discount display (if needed) |
| `deliverables` | array&lt;string&gt; | e.g., ["Preparation", "Action plan", "Resources"] |
| `isActive` | boolean | |
| `createdAt` | timestamp | |
| `updatedAt` | timestamp | |

### `bookings` (shared — your proposal is good)

Keep your `bookings` schema as-is. Add these fields:

| Field | Type | Notes |
|-------|------|-------|
| `consultationId` | string | ref to `consultations` |
| `tier` | string | denormalized from consultation |
| `paymentMethod` | string | `stripe`, `manual`, `invoice` |
| `invoiceUrl` | string? | for manual payments |

### `availability` (keep your proposal)

Keep your `availability` schema as-is.

---

## Answers to Web Team Questions

### 1. Should iOS display bookings?

**Yes.** iOS should show:
- Upcoming appointments (calendar view)
- Booking history
- Booking status (pending, confirmed, completed, cancelled)
- Payment status

### 2. Should iOS support booking natively?

**Phase 1:** Redirect to web booking page (simpler, faster)
**Phase 2:** Native booking with Stripe SDK (better UX)

Recommendation: Start with redirect, add native later if needed.

### 3. Collection naming convention

We're using camelCase in iOS (`consultations`, `bookings`, `availability`). Your snake_case proposal is fine — we'll map in iOS models.

### 4. Offer email flow notification in iOS

**Yes.** Add to `job_submissions`:
- `offerSentAt` (timestamp)
- `offerType` (string)
- `offerToken` (string for tracking)

iOS should show a banner/notification when an offer is sent.

### 5. Availability sync

**Proposal:** Web admin manages availability (writes), iOS reads it.
**Rationale:** Admin portal is web-based. iOS is display-only for now.

### 6. Stripe SDK vs redirect

**Phase 1:** Redirect to web booking page (Stripe Checkout)
**Phase 2:** If high demand, add native Stripe SDK to iOS

### 7. Currency conversion

Store both `priceEUR` and `priceUSD`. Display based on:
- User's locale (iOS)
- Audience type (student = EUR, company = USD)
- Manual toggle in settings

---

## Implementation Timeline (Unified)

| Phase | What | Owner |
|-------|------|-------|
| 1 | Reconcile collection schemas (consultations, bookings) | Both |
| 2 | Update pricing to €15 First Coffee, EUR for students | Web |
| 3 | Stripe Checkout implementation | Web |
| 4 | Manual payment option for companies | Web |
| 5 | Booking page (`/book`) with Apple theme | Web |
| 6 | iOS bookings display (read-only) | iOS |
| 7 | Assessment email after First Coffee | Web |
| 8 | Monthly community digest system | Web |
| 9 | Lead scoring system | iOS |
| 10 | Builder portfolio platform | Both |

---

## What We Need From Web Team

1. Confirm or adjust the unified `consultations` schema above
2. Update pricing to €15 First Coffee (remove ~~$15~~ $7 discount framing)
3. Add EUR pricing fields alongside USD
4. Add manual payment option for companies
5. Build Stripe Checkout + manual invoice flow
6. Update offer email to remove upsell pressure (focus on relationship)
7. Add `offerSentAt`, `offerType`, `offerToken` to `job_submissions`
8. Implement assessment email after First Coffee (with thoughtful questions)
9. Implement monthly community digest system

---

## What iOS Will Build

1. Bookings display (upcoming, history, status)
2. Lead scoring dashboard (9 dimensions)
3. Builder portfolio view (Apple-style)
4. CRM view updates (show lead scores, badges)
5. Offer notification in app
6. Consultation catalog display (read-only initially)

---

**Next Step:** Web team review and confirm unified approach, then we proceed with implementation.
