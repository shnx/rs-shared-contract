# iOS → Web: Paid Consultations, Quotations & Booking System — Proposal

**From:** iOS team
**To:** Web team
**Date:** July 7, 2026
**Re:** Next phase — paid consultations, quotation sending to companies/students, internal scheduling & booking

---

## Overview

Now that the email response system, CRM contacts, and transaction review workflow are implemented and synced, the next major feature is **paid consultations** — growing the platform from a training/CV center into a full consultation marketplace where:

1. **Companies** request consultations (training, advisory, workshops)
2. **Students/individuals** book paid 1-on-1 sessions
3. Platform handles **quotation generation, scheduling, booking, and payment** internally
4. Both iOS and web apps share the same data contracts

---

## Proposed Firestore Collections

### 1. `consultations` — Service catalog

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | doc id |
| `title` | string | |
| `description` | string | |
| `type` | string | `company`, `student`, `group` |
| `category` | string | `training`, `advisory`, `workshop`, `technical`, `career` |
| `durationMinutes` | number | |
| `price` | number | major units |
| `currency` | string | `JOD`, `USD` |
| `isActive` | boolean | |
| `createdBy` | string | admin uid |
| `createdAt` | timestamp | |
| `updatedAt` | timestamp | |

### 2. `consultation_requests` — Inbound requests

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | doc id |
| `consultationId` | string? | ref to catalog, or null for custom |
| `requesterType` | string | `company`, `student` |
| `requesterName` | string | |
| `requesterEmail` | string | |
| `requesterPhone` | string? | |
| `company` | string? | company name if company request |
| `position` | string? | contact role at company |
| `topic` | string | what they want |
| `details` | string | free-text requirements |
| `preferredDates` | array&lt;timestamp&gt; | |
| `estimatedParticipants` | number? | for group/workshop |
| `status` | string | `new`, `reviewed`, `quoted`, `accepted`, `declined`, `booked`, `completed` |
| `assignedTo` | string? | admin/consultant uid |
| `createdAt` | timestamp | |
| `updatedAt` | timestamp | |

### 3. `quotations` — Price quotes sent to requesters

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | doc id |
| `requestId` | string | ref to `consultation_requests` |
| `items` | array&lt;object&gt; | `{description, quantity, unitPrice, total}` |
| `subtotal` | number | |
| `discount` | number? | |
| `tax` | number? | |
| `total` | number | |
| `currency` | string | |
| `validUntil` | timestamp | quote expiry |
| `status` | string | `draft`, `sent`, `accepted`, `rejected`, `expired` |
| `sentAt` | timestamp? | |
| `respondedAt` | timestamp? | |
| `createdBy` | string | admin uid |
| `createdAt` | timestamp | |
| `notes` | string? | internal notes |

### 4. `bookings` — Scheduled sessions

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | doc id |
| `requestId` | string? | ref to request if originated from one |
| `quotationId` | string? | ref to accepted quotation |
| `consultationId` | string? | ref to catalog item |
| `clientName` | string | |
| `clientEmail` | string | |
| `clientPhone` | string? | |
| `clientType` | string | `company`, `student` |
| `company` | string? | |
| `topic` | string | |
| `consultantId` | string | who delivers the session |
| `consultantName` | string | |
| `scheduledAt` | timestamp | session start |
| `durationMinutes` | number | |
| `location` | string | `online`, `onsite`, `hybrid` |
| `meetingUrl` | string? | Zoom/Meet link if online |
| `locationAddress` | string? | if onsite |
| `status` | string | `scheduled`, `confirmed`, `completed`, `cancelled`, `no_show` |
| `price` | number | |
| `currency` | string | |
| `paymentStatus` | string | `unpaid`, `partial`, `paid`, `refunded` |
| `transactionId` | string? | ref to `ady_transactions` when paid |
| `reminderSentAt` | timestamp? | |
| `feedback` | string? | post-session feedback |
| `rating` | number? | 1-5 |
| `createdAt` | timestamp | |
| `updatedAt` | timestamp | |

### 5. `consultant_availability` — Time slots

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | doc id |
| `consultantId` | string | user uid |
| `consultantName` | string | |
| `slotStart` | timestamp | |
| `slotEnd` | timestamp | |
| `isBooked` | boolean | |
| `bookingId` | string? | ref to booking |
| `recurrence` | string? | `none`, `weekly`, `daily` |
| `createdAt` | timestamp | |

---

## Proposed Cloud Functions

1. **`sendQuotation`** — callable, takes `requestId` + quotation data, generates branded HTML email with quotation PDF attachment, sends to requester, creates `quotations` doc, updates `consultation_requests.status = quoted`.

2. **`captureQuotationResponse`** — public HTTP endpoint (like `captureEmailResponse`), linked from quotation email buttons. Accepts `quotationId` + `response` (accept/reject). Updates quotation status and creates booking if accepted.

3. **`sendBookingConfirmation`** — callable, takes `bookingId`, sends confirmation email to client + consultant with calendar invite (.ics attachment).

4. **`sendBookingReminder`** — scheduled function, runs hourly, finds bookings within next 24h that haven't had reminders sent, sends reminder emails.

5. **`generateQuotationPDF`** — callable, takes quotation data, generates branded PDF using the existing PDF report builder pattern.

---

## iOS Implementation Plan

### Phase 1: Models & Repos (this week)
- Add `Consultation`, `ConsultationRequest`, `Quotation`, `Booking`, `ConsultantAvailability` models to `Models.swift`
- Add repositories to `FirestoreRepository.swift`
- Add `displayStatus` and `statusTone` computed properties for each

### Phase 2: Admin UI (next week)
- **Consultation Catalog** — CRUD for service offerings (admin)
- **Requests Inbox** — list of consultation requests with status filters
- **Quotation Builder** — form to create quotations with line items, generates PDF
- **Booking Calendar** — schedule view showing upcoming sessions
- **Availability Manager** — consultants set available time slots

### Phase 3: Client-facing UI (week 3)
- **Request Consultation** form (for students via public site / app)
- **Quotation Viewer** — client sees quotation with accept/reject buttons
- **Booking List** — client sees their upcoming/past sessions
- **Payment Integration** — link booking to transaction when paid

### Phase 4: Notifications & Automation (week 4)
- Email notifications for request received, quotation sent, booking confirmed, reminder
- Push notifications on iOS for upcoming sessions
- Auto-create transaction when booking payment is confirmed

---

## Integration With Existing Systems

### CRM
- `consultation_requests` from companies should create `CRMContact` entries with `role = "company"`
- Status pipeline: `new → reviewed → quoted → accepted → booked → completed`
- Reuses the same `StatusPill` tones we already have

### Transactions
- When a booking is paid, auto-create a `Transaction` with:
  - `type = .revenue`
  - `source = "consultation"`
  - `category = "Consultation Revenue"`
  - Linked `bookingId` field (new optional on Transaction)
- This keeps financial reporting unified

### Email System
- Reuses the existing Resend + Cloud Functions email infrastructure
- Branded HTML templates matching the existing email design
- Interactive response buttons (same pattern as `captureEmailResponse`)

### Student Dashboard
- Students see a "Book a Consultation" card on their dashboard
- Their bookings appear in a "My Sessions" section

---

## Questions for Web Team

1. **Payment processing** — Should we integrate an online payment gateway (Stripe, Tap, CliQ) for consultation payments, or keep it as manual/offline payment with receipt upload?

2. **Calendar integration** — Should we sync bookings with Google Calendar / Apple Calendar via .ics exports, or build a fully internal calendar view first?

3. **Quotation PDF branding** — Should quotation PDFs use the same branding as the venture report PDFs, or a separate template?

4. **Multi-consultant support** — Do we need multiple consultants with individual availability, or is it a single-consultant operation for now?

5. **Recurring bookings** — Should we support recurring consultation sessions (e.g., weekly training for a company)?

6. **Collection naming** — Are you OK with unprefixed collection names (`consultations`, `bookings`, etc.) since these are platform-wide, not ADY-specific?

---

## What We Need From Web Team

1. Confirm or adjust the collection schemas above
2. Decide on payment processing approach
3. Build the Cloud Functions (`sendQuotation`, `captureQuotationResponse`, `sendBookingConfirmation`, `sendBookingReminder`)
4. Add a public-facing "Request Consultation" form on the website
5. Add `SWIFT_APP_INTEGRATION_GUIDE.md` Section 22 with the consultation system schemas and Swift code examples

---

*Reply by adding a new `.md` file to `shared/docs/` or comment on GitHub issue #1.*
