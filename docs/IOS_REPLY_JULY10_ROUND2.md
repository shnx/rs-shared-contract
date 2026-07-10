# iOS → Web: Reply to Full Sync — July 10 (Round 2)

**From:** iOS team
**To:** Web team
**Date:** July 10, 2026
**Re:** Responding to WEB_REPLY_IOS_JULY10.md, WEB_UPDATE_IOS_JULY10.md, and WEB_IOS_FULL_SYNC_JULY10.md

---

## Thanks for the comprehensive sync!

We've read all 3 docs. Here's our response and what we've already implemented.

---

## 1. ✅ Timeline Collection — Confirmed

We use `timeline`. You're renaming `timeline_entries` → `timeline`. No iOS action needed.

---

## 2. ✅ Gemini 2.5-flash — Both Aligned

We updated all iOS Gemini calls to `gemini-2.5-flash` (July 10). You fixed your last `gemini-2.0-flash-exp` reference. Both platforms are now on 2.5-flash.

---

## 3. ✅ Recurring Payment Fields — iOS is the Reference

Our Insights tab uses `isRecurring`, `recurrenceFrequency`, `recurrenceEndDate`, `recurrenceParentId`, `label`, `recipientName`, `source`. We're glad you're adding these to the web Transaction type. iOS implementation is the reference — match field names and types.

---

## 4. ✅ site_config — IMPLEMENTED ON iOS

We've built a **Website Control panel** in the iOS app (under Website Tools project → "Website" tab). It reads and writes these `site_config` documents in real-time:

### What's implemented:

| Feature | site_config doc | iOS reads | iOS writes |
|---------|----------------|-----------|------------|
| **Section visibility** | `website_sections` | ✅ | ✅ Toggle switches for News, Projects, Courses, Membership, Tutorials |
| **Discover photo** | `discover_page` | ✅ | ✅ Sliders for zoom, offset X, offset Y + reset button |
| **Calendar status** | `google_calendar` | ✅ | ✅ Connect/disconnect button, shows connection status |
| **Portal messages** | `portal_messages` collection | ✅ | ✅ Compose form with audience targeting, priority, expiry |
| **Time slot config** | `time_slot_config` collection | ✅ | ✅ Add/delete tier-based slots (day, hours, tier, duration, seats, price) |

### How it works:
- Admin opens Website Tools project → taps "Website" tab
- Gets a segmented control with 5 sections: Sections, Discover Page, Messages, Time Slots, Calendar
- All changes write to Firestore immediately with `merge: true`
- Web portal sees changes in real-time via existing `addSnapshotListener`

### What's NOT yet implemented (lower priority):
- `site_config/roles` — we'll add this to the user management screen later
- `latex_templates` — will add when we build a CV builder
- `student_crm` — will add when we build student management features
- `call_invitations` — will add with student portal Schedule tab
- `time_slot_bookings` — will add with student portal Schedule tab
- Real-time `addSnapshotListener` on site_config docs (currently using one-shot reads + pull-to-refresh; will upgrade to listeners)

---

## 5. Open Items from Your Docs — Our Answers

### From WEB_IOS_FULL_SYNC_JULY10.md:

| # | Question | iOS Answer |
|---|----------|------------|
| 6 | Roles: `investor` vs `client` mapping | We use `investor` on iOS. Suggest keeping both — `investor` for ADY financial stakeholders, `client` for service clients. They're different user types. |
| 7 | `service_offerings` → `consultations` migration | Keep `service_offerings` — it's more general. `consultations` is too narrow. No migration needed. |
| 8 | `ady_crm` vs `crm` | `ady_crm` is for ADY project leads. `crm_contacts` is for website/applicant CRM. Keep both — they serve different purposes. |
| 9 | Stripe vs Tap | We don't use Stripe on iOS. Tap for JOD is fine. If EUR payments are needed later, we'll revisit. |

### From WEB_IOS_SYNC_REQUEST_JULY9.md:
We already responded in `IOS_SYNC_RESPONSE_JULY10.md` with our complete collections audit, 3-project confirmation, and feature list. Please check that doc if you haven't.

---

## 6. Collections We're Now Using (Updated)

| Collection | Status | Purpose |
|-----------|--------|---------|
| `site_config` | ✅ NEW — reading/writing | Website configuration (sections, discover photo, calendar) |
| `portal_messages` | ✅ NEW — reading/writing | Admin messaging to portal users |
| `time_slot_config` | ✅ NEW — reading/writing | Tier-based time slot configuration |
| `ady_transactions` | Existing | ADY transactions + recurring payments |
| `systemUsers` | Existing | User accounts + employee picker |
| All other existing collections | Unchanged | See IOS_SYNC_RESPONSE_JULY10.md |

### Collections we'll add in future sprints:
| Collection | When |
|-----------|------|
| `student_crm` | Student management features |
| `call_invitations` | Student portal Schedule tab |
| `time_slot_bookings` | Student portal Schedule tab |
| `latex_templates` | CV builder (if we build one) |
| `site_config/roles` | User management enhancement |

---

## 7. Action Items Summary

### ✅ iOS has done:
1. Website Control panel with site_config read/write
2. Portal messages compose + list
3. Time slot config add/delete
4. Calendar status toggle
5. 3-project structure confirmed
6. Gemini 2.5-flash update
7. ADY Insights tab (category breakdown + recurring payments + AI detection)
8. Collections audit shared

### 📋 iOS will do (future sprints):
1. Add `student_crm`, `call_invitations`, `time_slot_bookings` models
2. Build student portal Schedule tab with seat-booking UI
3. Build student portal Inbox tab for messages
4. Add `addSnapshotListener` for real-time site_config sync
5. Read `site_config/roles` for shared role definitions
6. Read `latex_templates` for CV builder

### 📋 Web team should do:
1. Rename `timeline_entries` → `timeline` (you confirmed)
2. Delete duplicate "First Payment" period (`1764518075230bwtmp8tlg`)
3. Add recurring payment fields to web Transaction type
4. Build web Insights tab
5. Build `updateEnrollmentCount` Cloud Function
6. Add `planName` to `student_memberships` on create
7. Reply to GitHub issue #1

---

## 8. No Schema Changes from iOS

All new iOS features use existing Firestore collections and fields. The only new collections we're writing to are ones the web team already created (`site_config`, `portal_messages`, `time_slot_config`).

---

*Shared in `shared/docs/IOS_REPLY_JULY10_ROUND2.md`*
