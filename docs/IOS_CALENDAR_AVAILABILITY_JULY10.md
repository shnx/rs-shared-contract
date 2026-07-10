# iOS → Web: Google Calendar + Availability Sync — July 10

**From:** iOS team
**To:** Web team
**Date:** July 10, 2026
**Re:** iOS now writes to `availability` collection — please align

---

## What's New on iOS

### 1. Real Google Calendar OAuth
The Website Control panel → Calendar tab now triggers the **actual Google Sign-In OAuth flow** (not just a boolean toggle). When admin taps "Connect Google Calendar":
- Uses `GoogleCalendarService.connectGoogleCalendar()` 
- Requests `calendar` scope
- Fetches upcoming events (next 30 days)
- Writes `calendarConnected: true` + timestamp to `site_config/google_calendar`
- Disconnect signs out of Google and sets `calendarConnected: false`

### 2. Availability Management
New "Availability" tab in Website Control panel with two modes:

#### Manual Add
- Admin picks a date, start hour, end hour, slot duration
- Assigns to a service offering (from `service_offerings` collection) or "General"
- Writes to `availability` collection

#### Sync from Google Calendar
- Shows a week navigator (prev/next/today)
- Computes **free time slots** by subtracting busy Google Calendar events from working hours (9am–6pm, Sun–Thu, skipping Fri/Sat weekend)
- Admin selects which free slots to publish
- Assigns selected slots to a service offering
- Writes all selected slots to `availability` collection

### 3. `availability` Collection Schema (iOS writes)

```jsonc
{
  "id": "UUID string",
  "offeringId": "string?",        // ref to service_offerings doc, or null for General
  "startDate": "Timestamp",
  "endDate": "Timestamp",
  "slotDurationMinutes": "number",  // 30, 45, 60, 90, 120
  "isBooked": false,
  "bookedBy": null,                // email/user ID when booked
  "createdAt": "Timestamp"
}
```

**Please confirm this matches your `availability` collection schema.** If not, let us know and we'll adjust.

---

## What the Web Team Should Do

1. **Confirm `availability` schema** matches the above
2. **Read `availability` docs** on the Discover/booking page — filter by `offeringId` to show slots per service
3. **Filter by `isBooked == false`** to only show open slots
4. **Update `isBooked` and `bookedBy`** when a booking is confirmed on the web
5. **Read `site_config/google_calendar`** to show calendar connection status badge

---

## Collections iOS Now Reads/Writes

| Collection | Read | Write |
|-----------|------|-------|
| `site_config/website_sections` | ✅ | ✅ |
| `site_config/discover_page` | ✅ | ✅ |
| `site_config/google_calendar` | ✅ | ✅ |
| `portal_messages` | ✅ | ✅ |
| `time_slot_config` | ✅ | ✅ |
| `availability` | ✅ | ✅ NEW |
| `service_offerings` | ✅ NEW | — |

---

*Shared in `shared/docs/IOS_CALENDAR_AVAILABILITY_JULY10.md`*
