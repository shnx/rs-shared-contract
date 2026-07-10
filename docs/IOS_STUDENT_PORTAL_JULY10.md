# iOS → Web: Student Portal + Real-time Sync — July 10 (Round 3)

**From:** iOS team
**To:** Web team
**Date:** July 10, 2026
**Re:** Student Inbox, Student Schedule, real-time listeners, all remaining action items

---

## What's New on iOS

### 1. Student Inbox (portal_messages)
- **Where:** Student Dashboard (My Track) → Inbox section
- Reads `portal_messages` filtered by audience (all/students/candidates/specific)
- Shows unread indicator with priority color (urgent=red, important=orange, normal=blue)
- Tapping a message marks it as read (updates `readBy` array in Firestore)
- **Collections:** `portal_messages` (read + update `readBy`)

### 2. Student Schedule (time_slot_config + time_slot_bookings + call_invitations)
- **Where:** Student Dashboard (My Track) → Schedule section
- Reads `time_slot_config` to show available booking slots (filters out `unavailable` tier)
- Shows call invitation banner if student has an active `call_invitations` record
- Student can book a slot → writes to `time_slot_bookings`
- Auto-applies discount if student has a call invitation
- Updates `call_invitations.status` to `scheduled` when a slot is booked
- Shows "My Bookings" list with status badges
- **Collections:** `time_slot_config` (read), `time_slot_bookings` (read/write), `call_invitations` (read/update)

### 3. Real-time site_config Listeners
- Website Control panel now uses `addSnapshotListener` on:
  - `site_config/website_sections` — section toggles update in real-time
  - `site_config/discover_page` — photo config updates in real-time
  - `site_config/google_calendar` — calendar status updates in real-time
- If web team changes any config, iOS sees it immediately (no refresh needed)

---

## Complete Action Items Status

| # | Web Team Request | iOS Status |
|---|-----------------|------------|
| 1 | Add `site_config` collection | ✅ Done — read/write all 4 docs |
| 2 | Add `latex_templates` collection | ⏳ Deferred — will add with CV builder |
| 3 | Use `addSnapshotListener` for real-time sync | ✅ Done — 3 listeners active |
| 4 | Confirm `timeline` collection name | ✅ Confirmed — we use `timeline` |
| 5 | Consider building Discover page | ⏳ Low priority — web-only for now |
| 6 | Add `student_crm` collection | ⏳ Deferred — will add with student management |
| 7 | Add `call_invitations` collection | ✅ Done — read by student email, update status |
| 8 | Add `time_slot_config` collection | ✅ Done — admin configures, student reads |
| 9 | Add `time_slot_bookings` collection | ✅ Done — student books, writes to Firestore |
| 10 | Add `portal_messages` collection | ✅ Done — admin composes, student reads |
| 11 | Add Schedule tab to student portal | ✅ Done — in My Track |
| 12 | Add Inbox tab to student portal | ✅ Done — in My Track |
| 13 | Read `call_invitations` by student email | ✅ Done |
| 14 | Write to `time_slot_bookings` on booking | ✅ Done |
| 15 | Update `call_invitations` status to `scheduled` | ✅ Done |
| 16 | Read `portal_messages` filtered by audience | ✅ Done |
| 17 | Update `portal_messages.readBy` on read | ✅ Done |
| 18 | Read `site_config/discover_page` | ✅ Done |
| 19 | Read `site_config/website_sections` | ✅ Done |
| 20 | Read `site_config/google_calendar` | ✅ Done |
| 21 | Write to `site_config` from iOS | ✅ Done |
| 22 | Coffee chat price `originalPriceUSD: 17.5` | ⏳ Will check seed data |
| 23 | Mentorship unlock from `student_crm` | ⏳ Deferred with student_crm |

---

## Collections iOS Now Reads/Writes (Complete)

| Collection | Read | Write | Where |
|-----------|------|-------|-------|
| `site_config/website_sections` | ✅ | ✅ | Website Control |
| `site_config/discover_page` | ✅ | ✅ | Website Control |
| `site_config/google_calendar` | ✅ | ✅ | Website Control |
| `portal_messages` | ✅ | ✅ | Website Control (compose) + Student Inbox (read) |
| `time_slot_config` | ✅ | ✅ | Website Control (admin) + Student Schedule (read) |
| `time_slot_bookings` | ✅ | ✅ | Student Schedule |
| `call_invitations` | ✅ | ✅ | Student Schedule |
| `availability` | ✅ | ✅ | Website Control → Availability |
| `service_offerings` | ✅ | — | Website Control → Availability |
| `ady_transactions` | ✅ | ✅ | ADY tabs |
| `systemUsers` | ✅ | ✅ | Auth + employee management |
| `courses`, `enrollments`, `video_tutorials`, `student_memberships` | ✅ | — | Student Dashboard |
| `job_submissions` | ✅ | — | Student Dashboard + Submissions |
| `buildings`, `documents` | ✅ | ✅ | Riad Shannak |

### Still deferred (future sprints):
- `student_crm` — with student management features
- `latex_templates` — with CV builder
- `site_config/roles` — with user management enhancement

---

## What We Need From Web Team

1. **Confirm `availability` schema** — iOS writes `{id, offeringId, startDate, endDate, slotDurationMinutes, isBooked, bookedBy, createdAt}`. Does this match?
2. **Confirm `time_slot_bookings` schema** — iOS writes `{id, studentId, studentName, studentEmail, slotDate, slotStartHour, slotEndHour, tier, originalPriceJOD, finalPriceJOD, discountPercent, callInvitationId, status, notes, createdAt}`. Does this match?
3. **Read `time_slot_bookings`** on web to show booking confirmations
4. **Rename `timeline_entries` → `timeline`** (you confirmed, has it been done?)
5. **Delete duplicate "First Payment" period** (`1764518075230bwtmp8tlg`)

---

*Shared in `shared/docs/IOS_STUDENT_PORTAL_JULY10.md`*
