# Web → iOS: Update — New Features Built (July 10, 2026)

**From:** Web team (Mohammad Shannak)
**To:** iOS team
**Date:** July 10, 2026
**Re:** New collections, types, and features the iOS app needs to be aware of

---

## Summary

We've built a major new feature set for the student portal: **invite-to-call flow**, **separate student CRM**, and **tier-based seat-booking scheduling**. This introduces 4 new Firebase collections and several new types that the iOS app should align with.

We still haven't received your `IOS_SYNC_RESPONSE_JULY9.md` — please respond to that as well so we can reconcile collections and tab structure.

---

## New Collections (4)

### 1. `student_crm` — Student CRM (separate from client CRM)

Student-specific CRM contacts, created when an admin responds to a submission. This is **separate** from `crm_contacts` (which remains for client/applicant CRM).

```jsonc
{
  "id": "string",
  "submissionId": "string",
  "name": "string",
  "email": "string",
  "phone": "string?",
  "role": "string",
  "status": "invited | active | enrolled | alumni | inactive",
  "cvFileName": "string?",
  "cvBase64": "string?",
  "cvNotes": "string?",
  "accountCreated": "boolean",
  "accountId": "string?",
  "callInvitationId": "string?",       // ref to call_invitations
  "callType": "free | discounted?",
  "callDiscountPercent": "number?",
  "callStatus": "sent | scheduled | completed | expired?",
  "proposedCourses": ["string"],
  "enrolledCourses": ["string"],
  "emailSentAt": "Timestamp?",
  "responseMessage": "string?",
  "notes": "string?",
  "createdAt": "Timestamp",
  "updatedAt": "Timestamp?"
}
```

### 2. `call_invitations` — Call Invitation Records

Created when admin chooses "Invite to Call" while reviewing a submission. Tracks the invitation lifecycle and links to a booked slot.

```jsonc
{
  "id": "string",
  "submissionId": "string",
  "studentName": "string",
  "studentEmail": "string",
  "callType": "free | discounted",
  "discountPercent": "number",         // 0 if free, 10-100 if discounted
  "personalMessage": "string?",
  "status": "sent | scheduled | completed | expired",
  "bookedSlotId": "string?",           // ref to time_slot_bookings
  "createdAt": "Timestamp",
  "expiresAt": "Timestamp?"            // 30 days from creation
}
```

### 3. `time_slot_config` — Tier-Based Time Slot Configuration

Admin-defined time slot tiers per day of week. Drives the seat-booking UI in the student portal.

```jsonc
{
  "id": "string",
  "dayOfWeek": "number",               // 0 = Sunday, 6 = Saturday
  "startHour": "number",               // 0-23
  "endHour": "number",                 // 0-23
  "tier": "premium | regular | discounted | unavailable",
  "slotDurationMinutes": "number",     // 30, 45, 60, 90, 120
  "priceJOD": "number",                // 0 if unavailable or free
  "maxSeats": "number",                // 0 if unavailable
  "label": "string?",                  // e.g. "Evening Rush"
  "labelAr": "string?",                // e.g. "المساء"
  "createdAt": "Timestamp"
}
```

### 4. `time_slot_bookings` — Student Slot Bookings

Created when a student books a time slot from the seat-booking UI.

```jsonc
{
  "id": "string",
  "studentId": "string",
  "studentName": "string",
  "studentEmail": "string",
  "slotDate": "string",                // "2026-07-15" (ISO date)
  "slotStartHour": "number",
  "slotEndHour": "number",
  "tier": "premium | regular | discounted",
  "originalPriceJOD": "number",
  "finalPriceJOD": "number",           // after discount
  "discountPercent": "number",         // 0-100
  "callInvitationId": "string?",       // if booked via invitation
  "status": "pending | confirmed | completed | cancelled",
  "notes": "string?",
  "createdAt": "Timestamp"
}
```

---

## New Email Template: `invite_call`

Added to `SubmissionResponseType`: `'invite_call'`

When admin selects "Invite to Call" in the Submission Response UI:
1. A `call_invitation` record is created (free or discounted)
2. A `student_crm` record is created (separate from client CRM)
3. An email postcard is sent mentioning the call invitation with booking instructions
4. The email includes account credentials and a link to the portal Schedule tab
5. The postcard is bilingual (EN/AR) with gender-aware greeting

The email template lives in `src/utils/emailTemplates.ts` — same `generateResponseEmail()` function, with new `callInvitation` param:

```typescript
callInvitation?: {
  callType: 'free' | 'discounted';
  discountPercent: number;
}
```

---

## New Tab: `time-slot-config`

Added to Operations category. Admin page to configure tier-based time slots:
- Day of week selector
- Start/end hour
- Tier: premium, regular, discounted, unavailable
- Slot duration (30/45/60/90/120 min)
- Max seats per slot
- Price in JOD
- Optional EN/AR labels

**Tab count is now 27** (was 26).

---

## New Student Portal Tab: Schedule

The Student Portal now has 5 tabs (was 4):
1. Overview
2. My CV
3. Courses
4. **Schedule** (NEW) — seat-booking UI
5. Tools

### Schedule Tab Features:
- **Call invitation banner** — shows active invitation (free/discounted) with expiry
- **Week navigation** — browse upcoming weeks
- **Tier-colored slot grid** — premium (amber), regular (blue), discounted (green), unavailable (gray)
- **Seat availability** — shows "X/Y seats" per slot, disables full slots
- **Pricing display** — original price with strikethrough + discounted price
- **Booking confirmation** — price breakdown before confirming
- **My Bookings** — list of existing bookings with status badges
- **Auto-discount** — if student has a call invitation, discount is applied automatically

---

## Updated Types (for iOS reference)

### `SubmissionResponseType`
```typescript
'propose_course' | 'propose_service' | 'general_welcome' | 'invite_call' | 'rejection'
```

### `TimeSlotTier`
```typescript
'premium' | 'regular' | 'discounted' | 'unavailable'
```

### Course/Lecture pricing fields (added previously)
- `Course.packageDiscountPercent: number`
- `Lecture.price: number`
- `Lecture.isFree: boolean`

---

## Action Items for iOS Team

1. **Add 4 new collections** to your iOS models: `student_crm`, `call_invitations`, `time_slot_config`, `time_slot_bookings`
2. **Add Schedule tab** to student portal — seat-booking UI with tier-based slots
3. **Read `call_invitations`** by student email to show the invitation banner
4. **Read `time_slot_config`** to populate the slot grid
5. **Write to `time_slot_bookings`** when a student confirms a booking
6. **Update `call_invitations` status** to `scheduled` when a slot is booked
7. **Respond to `WEB_IOS_SYNC_REQUEST_JULY9.md`** — we still need your collections audit and tab list
8. **Update tab count** to 27 (added `time-slot-config`)

---

## What's Next (Web Side)

- Student portal inbox and site-wide messaging system (in progress)
- Placement test flow and lecture selection enrollment
- LaTeX compiler, public video access, and course catalog in student tools

---

*This document is shared in `shared/docs/WEB_UPDATE_IOS_JULY10.md`*
