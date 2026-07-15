# iOS → Web Team Sync — July 15

**From:** iOS team
**To:** Web team
**Date:** July 15, 2026
**Re:** Complete collections audit, availability schema mismatch, and request for iOS control parity

---

## 1. Complete iOS Collections List (by feature/tab)

### Main Tab Bar

| Tab | Collection | Read | Write | Notes |
|-----|-----------|------|-------|-------|
| Dashboard | `projects` | ✅ | — | 3 projects: ADY, Riad Shannak, Website Tools |
| Dashboard | `transactions` | ✅ | ✅ | Web (unprefixed) collection — primary |
| Dashboard | `ady_transactions` | ✅ | ✅ | Legacy fallback, same schema |
| Dashboard | `periods` | ✅ | — | Web (unprefixed) |
| Dashboard | `ady_periods` | ✅ | — | Legacy fallback |
| Dashboard | `periodSummaries` | ✅ | — | |
| Dashboard | `ady_pdf_meta` | ✅ | ✅ | Receipt analysis metadata |

### ADY Project Workspace

| Feature | Collection | Read | Write | Notes |
|---------|-----------|------|-------|-------|
| ADY Overview | `transactions` / `ady_transactions` | ✅ | ✅ | Dual-read for compatibility |
| ADY Periods | `periods` / `ady_periods` | ✅ | ✅ | Dual-read |
| ADY Documents | `documents` / `ady_documents` | ✅ | ✅ | Dual-read |
| ADY Payments | `ady_labels` | ✅ | ✅ | Spend label definitions |
| ADY Payments | `labels` | ✅ | ✅ | Web mirror of labels (self-heals) |
| ADY Insights | `transactions` | ✅ | ✅ | Category breakdown, recurring detection |
| ADY Insights | `systemUsers` | ✅ | — | Employee list for recurring payments |
| ADY Meetings | `ady_meetings` | ✅ | ✅ | Meeting notes per period |
| ADY Work Sessions | `ady_workSessions` | ✅ | ✅ | Employee clock in/out |
| ADY Employee Perms | `ady_employeePermissions` | ✅ | ✅ | Per-employee permissions |
| ADY Stores | `ady_stores` | ✅ | ✅ | |
| ADY Installments | `ady_installments` | ✅ | ✅ | |
| ADY CRM | `ady_crm` | ✅ | ✅ | |
| ADY Subscriptions | `ady_subscriptions` | ✅ | ✅ | |
| ADY Contracts | `ady_contracts` | ✅ | ✅ | |

### Website Tools Project

| Feature | Collection | Read | Write | Notes |
|---------|-----------|------|-------|-------|
| Submissions | `job_submissions` | ✅ | ✅ | Status updates (invited, contacted, etc.) |
| Submissions | `courses` | ✅ | — | For course assignment |
| CRM | `crm_contacts` | ✅ | ✅ | Via `ady_crm` prefixed + `crm_contacts` |
| Email Inbox | `email_inbox` | ✅ | ✅ | Email-to-expense system |
| Email Responses | `email_responses` | ✅ | — | Student interactive email replies |
| Careers | `career_opportunities` | ✅ | — | Job board |
| Consultations | `consultations` | ✅ | ✅ | Builder ecosystem |
| Builder Portfolios | `builder_portfolios` | ✅ | — | |
| Talent Matches | `talent_matches` | ✅ | — | |
| Monthly Content | `monthly_content` | ✅ | — | |
| Project Challenges | `project_challenges` | ✅ | — | |
| Milestones | `milestones` | ✅ | ✅ | AI-powered project tracking |
| Bookings | `bookings` | ✅ | ✅ | Shared with web — consultations |
| Service Offerings | `service_offerings` | ✅ | — | Read-only on iOS |
| Availability | `availability` | ✅ | ✅ | **Schema mismatch — see section 2** |
| Timeline | `timeline` | ✅ | ✅ | Renamed from `timeline_entries` |
| Venture Reports | `venture_reports` | ✅ | ✅ | |
| Shared Views | `sharedViews` | ✅ | ✅ | Shareable report links |

### Website Control Panel (iOS admin)

| Feature | Collection | Read | Write | Notes |
|---------|-----------|------|-------|-------|
| Section Toggles | `site_config/website_sections` | ✅ | ✅ | Real-time listener |
| Discover Photo | `site_config/discover_page` | ✅ | ✅ | Real-time listener, nested `photoPosition` |
| Calendar Status | `site_config/google_calendar` | ✅ | ✅ | Real-time listener, `connected`/`connectedAt` |
| Portal Messages | `portal_messages` | ✅ | ✅ | Compose + list, audience targeting |
| Time Slot Config | `time_slot_config` | ✅ | ✅ | Admin configures tiers, pricing, seats |
| Availability | `availability` | ✅ | ✅ | Manual add + Google Calendar sync |
| Service Offerings | `service_offerings` | ✅ | — | Read for availability assignment |

### Student Dashboard (My Track)

| Feature | Collection | Read | Write | Notes |
|---------|-----------|------|-------|-------|
| Inbox | `portal_messages` | ✅ | ✅ | Filtered by audience, marks `readBy` |
| Schedule | `time_slot_config` | ✅ | — | Filters out `unavailable` tier |
| Schedule | `time_slot_bookings` | ✅ | ✅ | Student books slots |
| Schedule | `call_invitations` | ✅ | ✅ | Read by email, update status to `scheduled` |
| Courses | `courses` | ✅ | — | Published courses |
| Enrollments | `enrollments` | ✅ | — | By studentId |
| Video Tutorials | `video_tutorials` | ✅ | — | |
| Memberships | `student_memberships` | ✅ | — | |
| Applications | `job_submissions` | ✅ | — | Student's own submissions |

### Riad Shannak Project

| Feature | Collection | Read | Write | Notes |
|---------|-----------|------|-------|-------|
| Properties | `businesses` | ✅ | ✅ | Real estate buildings |
| Transactions | `transactions` | ✅ | ✅ | Riad-specific transactions |
| Clients | `clients` | ✅ | — | |
| Buildings | `buildings` | ✅ | ✅ | |

### Settings / Auth

| Feature | Collection | Read | Write | Notes |
|---------|-----------|------|-------|-------|
| Users | `systemUsers` | ✅ | ✅ | Auth, roles, employee management |
| Projects | `project_members` | ✅ | — | User→Project membership |

### Collections iOS does NOT use yet (deferred)
- `student_crm` — deferred, will add with student management
- `latex_templates` — deferred, will add with CV builder
- `site_config/roles` — deferred, will add with user management enhancement
- `widget_configs` — web-only dashboard widgets
- `audit_log` — web-only action logging
- `tap_config` — web-only payment config
- `contact_submissions` — web-only contact form
- `membership_plans` — iOS reads `student_memberships` only
- `lectures`, `placement_tests`, `placement_test_results` — web-only
- `cv_templates`, `prepared_cvs` — web-only
- `quizzes` — web-only
- `student_invitations` — web-only

---

## 2. Availability Schema Mismatch — NEEDS RESOLUTION

### What iOS writes to `availability`:

```jsonc
{
  "id": "UUID string",
  "offeringId": "string?",        // ref to service_offerings, or null for General
  "startDate": "Timestamp",       // concrete date+time
  "endDate": "Timestamp",         // concrete date+time
  "slotDurationMinutes": "number",
  "isBooked": false,
  "bookedBy": null,
  "createdAt": "Timestamp"
}
```

### What the web canonical spec (`BOOKING_SYSTEM_CANONICAL.md`) defines:

```jsonc
{
  "id": "string",
  "dayOfWeek": "number",          // 0=Sunday..6=Saturday — RECURRING weekly
  "startTime": "string",          // "10:00" 24h format
  "endTime": "string",            // "18:00" 24h format
  "slotMinutes": "number?",       // default 30
  "bufferMinutes": "number?",     // default 15
  "maxBookingsPerDay": "number?", // default 5
  "timezone": "string?",          // default "Asia/Amman"
  "isActive": "boolean?",
  "createdAt": "Timestamp?"
}
```

### The problem:
- **iOS writes concrete time slots** (`startDate`/`endDate` timestamps) — one doc per bookable slot
- **Web canonical spec uses recurring weekly windows** (`dayOfWeek` + `startTime`/`endTime` strings) — client expands into slots
- These are fundamentally different models. We need to pick one.

### iOS preference:
We can adapt to the recurring weekly model if web confirms. iOS already has the logic to expand weekly windows into concrete slots (the "Sync from Google Calendar" feature does exactly this). We would need:
1. Confirmation that `dayOfWeek` + `startTime`/`endTime` is the canonical schema
2. Agreement on who writes — web admin only, or both platforms?
3. How `isBooked`/`bookedBy` works in the recurring model (is it on the `bookings` doc instead?)

**Please confirm which schema we should standardize on.**

---

## 3. Request: iOS Control Parity

The iOS app should have admin control over everything the web portal controls. Currently iOS has **read-only** access to several collections that web manages. We want to add write capability on iOS for:

### 3a. Service Offerings (`service_offerings`)
- **Current iOS:** Read-only
- **Web:** Full CRUD (create, edit, toggle `isActive`, reorder)
- **iOS request:** Add create/edit/toggle on iOS Website Control panel
- **Why:** Admin should be able to manage service catalog from mobile

### 3b. Bookings (`bookings`)
- **Current iOS:** Read + create (via student schedule)
- **Web:** Full management (confirm, cancel, mark completed)
- **iOS request:** Add booking management view on iOS — confirm/cancel/complete
- **Why:** Admin needs to manage bookings from mobile

### 3c. CRM Contacts (`crm_contacts`)
- **Current iOS:** Read via `ady_crm` (prefixed)
- **Web:** Full CRUD on `crm_contacts`
- **iOS request:** Align on single collection name — should we both use `crm_contacts` (unprefixed)?
- **Why:** Currently iOS reads `ady_crm` and web reads `crm_contacts` — potential data split

### 3d. Email Inbox (`email_inbox`)
- **Current iOS:** Read-only (views pending emails)
- **Web:** Full management (review, approve, reject email-to-expense)
- **iOS request:** Add approve/reject actions on iOS
- **Why:** Admin should process emails from mobile

### 3e. Contact Submissions (`contact_submissions`)
- **Current iOS:** Not used
- **Web:** Full management (reply, mark resolved)
- **iOS request:** Add read + reply on iOS
- **Why:** Admin should see and reply to contact form submissions

### 3f. Membership Plans (`membership_plans`)
- **Current iOS:** Not used (reads `student_memberships` only)
- **Web:** Full CRUD
- **iOS request:** Add read access on iOS student dashboard
- **Why:** Students should see available plans

### 3g. Courses / Lectures (`courses`, `lectures`)
- **Current iOS:** Read-only (`courses`)
- **Web:** Full CRUD (courses, lectures, placement tests)
- **iOS request:** Add course management on iOS — at minimum toggle `isPublished`, edit title/description
- **Why:** Admin should manage courses from mobile

### 3h. Video Tutorials (`video_tutorials`)
- **Current iOS:** Read-only
- **Web:** Full CRUD
- **iOS request:** Add upload/edit/toggle on iOS
- **Why:** Admin should manage video content from mobile

### 3i. Tap Config (`tap_config`)
- **Current iOS:** Not used
- **Web:** Full management (payment gateway keys)
- **iOS request:** Add read-only on iOS (to show payment status)
- **Why:** Admin should see if payments are configured

### 3j. Audit Log (`audit_log`)
- **Current iOS:** Not used
- **Web:** Read
- **iOS request:** Add read-only on iOS
- **Why:** Admin should be able to review recent actions

### 3k. Widget Configs (`widget_configs`)
- **Current iOS:** Not used
- **Web:** Full CRUD
- **iOS request:** Add read on iOS dashboard
- **Why:** Dashboard widgets should sync across platforms

### 3l. Project Members (`project_members`)
- **Current iOS:** Read-only
- **Web:** Full CRUD
- **iOS request:** Add assign/remove members on iOS
- **Why:** Admin should manage project membership from mobile

---

## 4. Questions for Web Team

1. **Availability schema** — Should we standardize on the recurring weekly model (`dayOfWeek`/`startTime`/`endTime`) from `BOOKING_SYSTEM_CANONICAL.md`, or the concrete slot model (`startDate`/`endDate`) that iOS currently writes? We need a definitive answer.

2. **CRM collection naming** — iOS uses `ady_crm` (prefixed), web uses `crm_contacts`. Should we consolidate to `crm_contacts` only? If so, we need a migration plan for existing `ady_crm` docs.

3. **`service_offerings` schema** — The remote branch has `schemas/service-offering.json` with fields like `billing`, `originalPriceUSD`, `priceMaxUSD`, `requiresScheduling`, `features`, `featuresAr`, `sortOrder`. iOS currently only reads `name`, `nameAr`, `type`, `audience`, `priceUSD`, `description`, `isActive`. Should we expand the iOS model to match?

4. **`bookings` schema** — The canonical spec has `serviceType` enum: `coffee_time`, `curriculum_builder`, `course_access`, `company_assessment`. iOS `Booking` model has `serviceType` as a free string and also has `consultationId`, `tier`, `priceEUR`, `paymentMethod`, `invoiceUrl` fields not in the canonical schema. Should we align to the canonical schema and drop the extra fields?

5. **Booking pricing** — Canonical spec says USD only (`priceUSD`). iOS `TimeSlotBooking` uses JOD (`originalPriceJOD`, `finalPriceJOD`). Are these two separate booking systems, or should they be unified?

6. **`timeline` vs `timeline_entries`** — Web confirmed rename to `timeline`. Has this been done in the web codebase? iOS `TimelineRepository` still uses `timeline_entries` as the collection path — we need to update this.

7. **`syncBookingToCalendar` Cloud Function** — Is this deployed? iOS asked about this in `IOS_FIREBASE_REPLY_JULY10.md` but haven't received confirmation. We need to know if we should call it or use client-side OAuth.

8. **Stripe vs Tap** — Canonical spec mentions Stripe Checkout. Web previously mentioned Tap gateway. Which payment provider are we standardizing on?

9. **Duplicate "First Payment" period** — Web was supposed to delete `1764518075230bwtmp8tlg` from `ady_periods`. Has this been done? iOS still sees it.

10. **`updateEnrollmentCount` Cloud Function** — Was this built? iOS needs it to keep `course.studentCount` accurate.

---

## 5. iOS Features Web May Not Know About

### ADY Insights Tab (built July 10)
- Category breakdown with spending visualization
- Recurring payments management (add, toggle, AI detect)
- AI-powered recurring detection using `gemini-2.5-flash`
- Uses existing `Transaction` fields — no schema changes
- Collections: `transactions`, `systemUsers`

### Meeting Notes (ady_meetings)
- Per-period meeting notes with agenda items
- iOS-only feature, uses `ady_meetings` collection
- Fields: `periodId`, `title`, `agenda`, `notes`, `attendees`, `meetingDate`

### Work Sessions (ady_workSessions)
- Employee clock in/out tracking
- Weekly timesheet view
- Uses `ady_workSessions` collection
- Fields: `userId`, `startTime`, `endTime`, `duration`, `projectId`, `note`

### Employee Permissions (ady_employeePermissions)
- Per-employee granular permissions
- Uses `ady_employeePermissions` collection
- Fields: `userId`, `canViewTransactions`, `canAddTransactions`, `canManagePeriods`, etc.

### Google Calendar Integration (iOS)
- Real OAuth flow via Google Sign-In
- Fetches events for next 30 days
- Computes free slots by subtracting busy events from working hours (9am-6pm, Sun-Thu)
- Admin can publish free slots to `availability` collection
- Writes `connected`/`connectedAt` to `site_config/google_calendar`

### Milestones
- AI-powered project milestone tracking
- Uses `milestones` collection
- Fields: `title`, `category`, `status`, `date`, `tags`, `aiGenerated`, `relatedIds`

---

## 6. Summary of Action Items

### Web team needs to:
1. **Confirm availability schema** — recurring weekly vs concrete slots
2. **Confirm CRM collection** — consolidate to `crm_contacts`?
3. **Confirm `service_offerings` schema** — should iOS expand its model?
4. **Confirm `bookings` schema** — align to canonical or keep extended?
5. **Confirm pricing currency** — USD only or JOD+USD?
6. **Rename `timeline_entries` → `timeline`** in web codebase if not done
7. **Confirm `syncBookingToCalendar` deployment status**
8. **Confirm Stripe vs Tap** as payment provider
9. **Delete duplicate "First Payment" period** if not done
10. **Confirm `updateEnrollmentCount` Cloud Function** status
11. **Grant iOS write access** to `service_offerings`, `bookings` management, `crm_contacts`, `contact_submissions`, `courses`, `video_tutorials`, `project_members`
12. **Share `BOOKING_SYSTEM_CANONICAL.md`** on main branch (currently only on a dev branch)

### iOS team will:
1. Update `TimelineRepository` to use `timeline` instead of `timeline_entries`
2. Expand `ServiceOffering` model to match canonical schema (pending web confirmation)
3. Add admin CRUD views for service offerings, bookings, courses, video tutorials
4. Add read access for `contact_submissions`, `membership_plans`, `tap_config`, `audit_log`, `widget_configs`
5. Add project member management on iOS
6. Align availability schema once web confirms direction
7. Consolidate CRM to `crm_contacts` if web confirms

---

*Shared in `shared/docs/IOS_WEB_SYNC_JULY15.md`*
