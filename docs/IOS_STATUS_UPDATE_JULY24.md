# iOS → Web Team: Status Update — July 24

**From:** iOS team
**To:** Web team
**Date:** July 24, 2026
**Re:** iOS action items completed, pending web team responses, and new features

---

## 1. iOS Action Items — COMPLETED since July 15 sync

### ✅ Full Admin Portal (AdminPortalView.swift)
All 10 admin sections implemented with full CRUD:
- **Dashboard** — today's bookings, upcoming, revenue, default price
- **Bookings** — search/filter, confirm/cancel/complete, session link editing
- **Calendar** — recurring weekly availability rules, calendar blocks, time slot config
- **Payments** — default booking price editor, Tap config (test/production)
- **Coupons** — list, create with random code, copy, toggle active
- **People** — Users, CRM Contacts, Student CRM, Job Submissions, Call Invitations
- **Services** — service offerings CRUD, price escalation rules
- **Courses** — Courses, Membership Plans, Video Tutorials, Skill Questions
- **Website** — Site Sections, Contact Submissions, Careers, Portal Messages, User Messages
- **Settings** — Role definitions, Analytics, Agent Log

### ✅ New Swift Models (AdminModels.swift)
18 new Codable models with optional fields for safe Firestore decoding:
- `PaymentConfig`, `TapConfig`, `CalendarBlock`, `Coupon`, `SessionOutcome`
- `PriceEscalationRule`, `ContactSubmission`, `UserMessage`, `SkillQuestion`
- `DiscoverQuizResult`, `AnalyticsEvent`, `AnalyticsSession`, `AgentLog`
- `MembershipPlan`, `RoleDefinition`, `ServiceOfferingFull`, `BookingFull`
- `StudentCRM`, `AvailabilityRule` (recurring weekly), `AnyCodable`

### ✅ New Firestore Repositories (AdminRepositories.swift)
18 repository classes with custom query methods for all new collections.

### ✅ FirestoreRepository Enhancement
- Added `add()` method for auto-generated document IDs
- Added `firestore` and `path` computed properties for subclass access

### ✅ Availability Schema — Aligned to Web Canonical
iOS now uses the recurring weekly model (`dayOfWeek`, `startTime`, `endTime`) matching `BOOKING_SYSTEM_CANONICAL.md`. The old concrete slot model (`startDate`/`endDate`) is no longer written.

### ✅ Timeline Collection Rename
`TimelineRepository` primary collection changed from `timeline_entries` to `timeline`. Still reads `timeline_entries` as fallback for backward compatibility.

### ✅ ServiceOffering Model Expanded
`ServiceOfferingFull` model includes all web schema fields: `billing`, `originalPriceUSD`, `priceMaxUSD`, `requiresScheduling`, `features`, `featuresAr`, `sortOrder`, etc.

### ✅ UserPermissions — 14 Fields
Already includes `canManageSubscriptions`, `canGenerateReports`, `canComment`, `canManageProjects`.

### ✅ UserRole — manager role
Already includes `manager` in the enum.

### ✅ Public Booking Search + Registration
New "حجوزاتي" (My Bookings) section on the public site:
- Visitors search by name (or partial) + email
- `BookingsRepository.searchByNameAndEmail()` — fetches by email, filters by partial name
- Found booking → user can register an account (username + password)
- `AuthManager.registerFromBooking()` — creates Firebase Auth user, SystemUser doc, links bookings via `userId`

---

## 2. STILL PENDING — Web Team Responses Needed

These 10 questions from `IOS_WEB_SYNC_JULY15.md` are still unanswered:

| # | Question | Impact |
|---|----------|--------|
| 1 | **Availability schema** — confirm recurring weekly is canonical | iOS already adapted, need confirmation |
| 2 | **CRM collection** — consolidate to `crm_contacts`? | iOS reads both `ady_crm` and `crm_contacts` |
| 3 | **`service_offerings` schema** — confirm full schema | iOS already expanded model |
| 4 | **`bookings` schema** — align to canonical or keep extended? | iOS has extra fields (consultationId, tier, priceEUR) |
| 5 | **Pricing currency** — USD only or JOD+USD? | Two booking systems with different currencies |
| 6 | **`timeline` rename** — done in web codebase? | iOS renamed primary, keeping fallback |
| 7 | **`syncBookingToCalendar`** — deployed? | iOS uses client-side OAuth instead |
| 8 | **Stripe vs Tap** — which payment provider? | iOS admin shows Tap config |
| 9 | **Duplicate "First Payment" period** — deleted? | iOS still sees it |
| 10 | **`updateEnrollmentCount`** — built? | iOS needs it for course student counts |

---

## 3. New iOS Features Web Should Know About

### Public Booking Search (July 15)
- New public site section "حجوزاتي" — unauthenticated users can search bookings by name + email
- Registration flow: found booking → create account → bookings linked via `userId`
- `AuthManager.registerFromBooking()` creates Firebase Auth user + SystemUser doc + updates all bookings with that email to set `userId`

### Admin Portal (July 15)
- Full admin portal accessible via shield tab in MainTabView (admin role only)
- All web admin controls mirrored in native SwiftUI
- Uses `getAllSafe()` for tolerant Firestore decoding (skips malformed docs)

---

## 4. No Schema Changes from iOS Side

iOS has not changed any Firestore schemas. All changes are iOS-side only (new models, new views, new repositories). The only Firestore write change is:
- `TimelineRepository` now writes to `timeline` instead of `timeline_entries`
- `AuthManager.registerFromBooking()` writes `userId` to existing `bookings` docs when a user registers from a booking lookup

---

## TL;DR

1. ✅ All iOS action items from July 15 sync are complete
2. ⚠️ 10 questions still pending web team response — please reply in `shared/docs/` or GitHub issue #3
3. ✅ Admin portal fully implemented with all web controls
4. ✅ Public booking search + registration flow added
5. ✅ Timeline collection renamed to `timeline`
6. ✅ Availability schema aligned to recurring weekly model

*Shared in `shared/docs/IOS_STATUS_UPDATE_JULY24.md`*
