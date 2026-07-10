# Web → iOS: Reply to July 10 Sync — July 10

**From:** Web team (Mohammad Shannak)
**To:** iOS team
**Date:** July 10, 2026
**Re:** Responding to IOS_SYNC_RESPONSE_JULY10.md and IOS_WEB_SYNC_JULY10.md

---

## Thanks

Great work on the 3-project structure, Insights tab, and cleanup. We've read both your docs carefully.

---

## 1. Timeline Collection — RESOLVED

You're right — there's a naming mismatch:

| Platform | Collection | Status |
|----------|-----------|--------|
| Web | `timeline_entries` | Active — used in `database.ts` and `sharedBudgetClient.ts` |
| iOS | `timeline` | Active |

**Resolution:** We'll rename our collection to `timeline` to match iOS. This is a simple rename on our side — we'll update `STORAGE_KEYS.TIMELINE` from `'timeline_entries'` to `'timeline'` and migrate any existing docs.

**Action (web):** Rename `timeline_entries` → `timeline` in `database.ts` and `sharedBudgetClient.ts`.
**Action (iOS):** None — you already use `timeline`.

---

## 2. Gemini Model — FIXED

We found one remaining `gemini-2.0-flash-exp` reference in `Transactions.tsx` (the AI chat feature). **Updated to `gemini-2.5-flash`**.

Our Cloud Functions already use `gemini-2.5-flash` — no issue there.

**Status:** ✅ Fixed.

---

## 3. Recurring Payment Fields — Confirmed

We see your Insights tab uses these fields:
- `isRecurring: Bool?`
- `recurrenceFrequency: String?`
- `recurrenceEndDate: Date?`
- `recurrenceParentId: String?`
- `label: String?`
- `recipientName: String?`
- `source: String?`

**Web status:** Our Transaction type doesn't have these fields yet. We'll add them to the web Transaction interface so web-created transactions can also be recurring. We'll build a web Insights view in the next sprint.

**Action (web):** Add recurring fields to Transaction type and build Insights tab.
**Action (iOS):** None — your implementation is the reference.

---

## 4. Discover Page + Booking — Web-Only (Agreed)

You said: *"The Discover/booking system is web-only for now. If iOS needs it later, we'll align with your schemas."*

**Agreed.** However, we've now centralized the Discover page config in Firestore so if you ever build a Discover page, the data is ready:

- `site_config/discover_page` — photo position (zoom/pan)
- `site_config/website_sections` — section visibility toggles
- `service_offerings` — pricing and packages (already in Firestore)

No action needed from iOS now.

---

## 5. Centralized Firestore Config — NEW (please read)

We've migrated all shared config from localStorage to Firestore so **both platforms can read/write and see changes in real-time**.

### New collection: `site_config`

| Document ID | What it controls | iOS should |
|-------------|-----------------|------------|
| `discover_page` | Photo zoom/pan | Read on Discover page if you build one |
| `website_sections` | Show/hide news, projects, courses, etc. | Read on app launch for section visibility |
| `google_calendar` | Calendar connection status | Read to show "connected" badge |
| `roles` | Role definitions + permissions | Read for user management |

### New collection: `latex_templates`

Custom CV templates — iOS can read these for a CV builder if you build one.

### Swift pattern for reading site_config:

```swift
// Read
let doc = try await db.collection("site_config").document("website_sections").getDocument()
let showNews = doc.data()?["showNews"] as? Bool ?? false

// Listen for real-time changes
db.collection("site_config").document("website_sections").addSnapshotListener { snapshot, error in
    guard let data = snapshot?.data() else { return }
    // Update UI
}

// Write (when admin changes something on iOS)
try await db.collection("site_config").document("website_sections").setData([
    "showNews": true,
    "showProjects": true,
    // ...
], merge: true)
```

**Full schema details:** See `WEB_IOS_FULL_SYNC_JULY10.md` in `shared/docs/`.

**Action (iOS):** Add `site_config` and `latex_templates` to your Firestore models. Use `addSnapshotListener` for real-time sync.

---

## 6. Your Open Items from July 6 — Status

| # | Item | Status |
|---|------|--------|
| 1 | Delete duplicate "First Payment" period | ⚠️ Still pending — will delete `1764518075230bwtmp8tlg`, keep `CBE91C2E-...` |
| 2 | Update `isExpense` for freelance/salary | ✅ Done — web treats freelance/salary as expenses |
| 3 | Add `repliedAt` to `contact_submissions` | ✅ Done |
| 4 | `extractCVData` Cloud Function | ✅ Done — deployed |
| 5 | `sendContactReply` Cloud Function | ✅ Done — deployed |
| 6 | Reply to GitHub issue #1 | Will do today |

---

## 7. iOS Open Items from NOTES.md — Our Response

| # | Item | Web Response |
|---|------|-------------|
| 1 | Unified CRM — merge `crm` and `crm_contacts` | We use `ady_crm` for leads and `crm_contacts` for submissions. Recommend keeping both with a `source` field. No merge needed. |
| 2 | Enrollment count sync — Cloud Function | Will build `updateEnrollmentCount` Cloud Function that updates `course.studentCount` on enrollment create/delete. |
| 3 | Membership plan name — denormalize | ✅ Agree — will add `planName` to `student_memberships` on create. |
| 4 | Video tutorial ordering | We sort by `order` field, fallback to `createdAt`. iOS should do the same. |

---

## 8. Collections iOS Uses That We Now Know About

| Collection | Our plan |
|-----------|----------|
| `work_sessions` | iOS-only — no web action needed |
| `employee_permissions` | iOS-only — no web action needed |
| `timeline` | ✅ Will rename from `timeline_entries` to match |

---

## 9. Updated Collections Count

Web now has these Firestore collections:

**Existing (shared):** `projects`, `transactions`, `periods`, `periodSummaries`, `documents`, `ady_transactions`, `ady_periods`, `ady_documents`, `ady_crm`, `crm_contacts`, `systemUsers`, `job_submissions`, `contact_submissions`, `courses`, `enrollments`, `video_tutorials`, `video_assignments`, `student_memberships`, `career_opportunities`, `buildings`, `timeline` (renamed), `email_responses`, `email_inbox`, `sharedViews`, `venture_reports`, `service_offerings`, `bookings`, `availability`, `tap_config`, `widget_configs`, `audit_log`

**New (shared):** `student_crm`, `call_invitations`, `time_slot_config`, `time_slot_bookings`, `portal_messages`, `site_config`, `latex_templates`

**Total: ~37 collections**

---

## 10. Action Items Summary

### Web team will do:
1. Rename `timeline_entries` → `timeline`
2. Delete duplicate "First Payment" period from Firestore
3. Add recurring payment fields to web Transaction type
4. Build web Insights tab (category breakdown + recurring payments)
5. Build `updateEnrollmentCount` Cloud Function
6. Add `planName` to `student_memberships` on create
7. Reply to GitHub issue #1

### iOS team should do:
1. Add `site_config` collection — read `website_sections`, `google_calendar`, `roles` docs
2. Add `latex_templates` collection — read for CV builder if applicable
3. Use `addSnapshotListener` on `site_config` docs for real-time sync
4. Confirm `timeline` collection name (we're aligning to your name)
5. Consider building Discover page using `service_offerings` + `site_config/discover_page` (optional, low priority)

### No action needed (aligned):
- 3-project structure ✅
- Gemini model ✅ (both on 2.5-flash)
- Duplicate collections / prefix logic ✅
- Discover page = web-only ✅
- Recurring payment fields = existing schema ✅

---

*Reply by adding a new `.md` file to `shared/docs/` or comment on GitHub issue #1.*
