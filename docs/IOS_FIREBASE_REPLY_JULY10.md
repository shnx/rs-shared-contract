# iOS → Web: Firebase Coordination Reply — July 10

**From:** iOS team
**To:** Web team
**Date:** July 10, 2026
**Re:** Reply to `WEB_IOS_FIREBASE_COORDINATION_JULY10.md` — schema alignment confirmed, all action items done

---

## Schema Alignment — FIXED

We've updated our Firestore field names to match your spec exactly:

### `site_config/discover_page`
- ✅ Now uses nested `photoPosition: { scale, offsetX, offsetY }` (was flat `photoScale`, `photoOffsetX`, `photoOffsetY`)
- ✅ Uses `updatedAt` (was `photoUpdatedAt`)

### `site_config/google_calendar`
- ✅ Now uses `connected` (was `calendarConnected`)
- ✅ Now uses `connectedAt` (was `calendarConnectedAt`)

### `site_config/website_sections`
- ✅ Now writes `updatedAt` and `updatedBy: "ios"` (was `photoUpdatedAt` only)

### `portal_messages`
- ✅ `readBy` is now `[String]?` (optional) to handle docs without the field
- ✅ `markAsRead` now adds student **email** to `readBy` array (was incorrectly adding message ID — bug fixed)

---

## iOS Action Items — Status

| # | Your Request | Status |
|---|-------------|--------|
| 1 | Add `site_config` listeners | ✅ Done — 3 real-time listeners (sections, discover, calendar) |
| 2 | Add `latex_templates` | ⏳ Deferred — will add with CV builder |
| 3 | Read `student_crm` for mentorship unlock | ⏳ Deferred — will add with student management |
| 4 | Use `syncBookingToCalendar` Cloud Function | ⏳ Need details — see below |
| 5 | Write calendar status to `site_config/google_calendar` | ✅ Done |
| 6 | Confirm `buildings` schema | ✅ Confirmed — matches your spec |
| 7 | `onSnapshot` on shared collections | ✅ Done for site_config |

---

## Questions About `syncBookingToCalendar` Cloud Function

We see you built a Cloud Function. We need to know:

1. **Is it deployed?** We can't call it until it's live.
2. **What's the function name?** You showed `syncBookingToCalendar` — confirm?
3. **Should iOS stop using direct Google OAuth for calendar events?** We currently use `GoogleCalendarService` with GoogleSignIn SDK for the admin's personal calendar. Should we:
   - **Keep OAuth** for admin calendar management (reading free/busy, creating events)
   - **Use Cloud Function** for booking confirmations (when a student books a slot)
   - This seems like the right split — let us know.

4. **For booking flow:** We write to `time_slot_bookings` when a student books. Should we also call the Cloud Function to create a calendar event for the booking? If so, we need the function deployed first.

---

## What iOS Now Has (Complete)

### Student Dashboard (My Track) — 7 sections visible:
1. **Inbox** — reads `portal_messages`, filtered by audience, marks as read
2. **Schedule** — reads `time_slot_config`, shows `call_invitations` banner, writes `time_slot_bookings`, updates invitation status
3. **My Courses** — reads `enrollments` + `courses`
4. **Available Courses** — reads `courses` where `isPublished`
5. **Video Tutorials** — reads `video_tutorials`
6. **Membership** — reads `student_memberships`
7. **My Application** — reads `job_submissions`

### Website Control Panel (Admin) — 6 tabs:
1. **Sections** — toggle website sections, real-time sync
2. **Discover** — photo zoom/pan, real-time sync
3. **Messages** — compose portal messages
4. **Time Slots** — configure tier-based slots
5. **Calendar** — real Google OAuth, upcoming events
6. **Availability** — sync free time from Google Calendar, assign to service offerings, publish to `availability` collection

### Real-time Listeners Active:
- `site_config/website_sections` ✅
- `site_config/discover_page` ✅
- `site_config/google_calendar` ✅

---

## Confirmed Agreements

| Item | Status |
|------|--------|
| `freelance`/`salary` = expense | ✅ Agreed |
| `timeline` collection name | ✅ Agreed |
| Gemini `gemini-2.5-flash` | ✅ Agreed |
| Booking flow: create → sync calendar → send email | ✅ Agreed |
| `source` field: `"ios"` or `"web"` | ✅ Agreed |
| Recurring payment fields | ✅ Agreed |

---

*Reply by adding a new `.md` file to `shared/docs/` or comment on GitHub.*
