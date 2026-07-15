# Changelog

All schema changes to the shared contract between iOS and web apps.

## [1.0.0] - 2026-07-01

### Added
- Initial shared contract extracted from iOS app.
- `transactions` / `ady_transactions` document schema.
- `periods` / `ady_periods` document schema.
- `documents` / `ady_documents` document schema.
- `periodSummaries` document schema.
- `ady_pdf_meta` document schema.
- `sharedViews` document schema (new — for shareable budget dashboard links).
- Two-tier category taxonomy (6 broad categories with sub-categories).
- `SHARED_LINK_GUIDE.md` — full guide for the shareable link feature.
- `DATA_CONTRACT.md` — complete Firestore reference.
- `CATEGORY_TAXONOMY.md` — category system documentation.

## [1.1.0] - 2026-07-02

### ⚠️ Breaking Changes
- **`systemUsers.permissions` expanded from 10 to 14 fields.** Added: `canManageSubscriptions`, `canGenerateReports`, `canComment`, `canManageProjects`. iOS `UserPermissions` struct must be updated.
- **`systemUsers.role` now includes `manager`.** Update role enum/switch on iOS.
- **`sharedViews` Firestore rules changed from server-only read to public read.** The web app now fetches shared budget data client-side (no Express endpoint). See updated `SHARED_LINK_GUIDE.md`.

### Added
- `venture_reports` collection schema (see `VENTURE_REPORT_WEB_HANDOFF.md`).
- `timeline` collection — public read for achievement photos on venture reports.
- Email system via Firebase Cloud Functions + Resend SMTP (`EMAIL_SYSTEM_SETUP.md`).
- `PORTAL_STRUCTURE_AND_SYNC.md` — full portal tab structure and sync proposal.
- `WEB_STATUS_UPDATE_2.md` — shared link fixes, Firestore rules, CV feedback system.
- Testing Lab page for admin testing (emails, fake users, flows).

### Changed
- `SHARED_LINK_GUIDE.md` — updated to reflect client-side rendering instead of Express endpoint.
- Category keywords in web `sharedBudgetClient.ts` synced to match `CATEGORY_TAXONOMY.md` exactly.
- Cloud Functions migrated from v1 to v2 SDK with `defineSecret()` for SMTP credentials.
- Sender email updated to `info@the-rs.com` (Resend verified domain).

## [1.5.0] - 2026-07-15

### Added
- **`IOS_WEB_SYNC_JULY15.md`** — comprehensive iOS→Web sync document with complete collections audit, availability schema mismatch report, and request for iOS control parity.
- **`ady_meetings` collection** — per-period meeting notes (iOS-only). Fields: `periodId`, `title`, `agenda`, `notes`, `attendees`, `meetingDate`.
- **`ady_workSessions` collection** — employee clock in/out tracking (iOS-only). Fields: `userId`, `startTime`, `endTime`, `duration`, `projectId`, `note`.
- **`ady_employeePermissions` collection** — per-employee granular permissions (iOS-only). Fields: `userId`, `canViewTransactions`, `canAddTransactions`, `canManagePeriods`, etc.
- **`milestones` collection** — AI-powered project milestone tracking. Fields: `title`, `category`, `status`, `date`, `tags`, `aiGenerated`, `relatedIds`.
- **Google Calendar OAuth integration (iOS)** — real Google Sign-In flow, fetches events, computes free slots, publishes to `availability` collection.

### ⚠️ Schema Conflicts Identified
- **`availability` collection** — iOS writes concrete time slots (`startDate`/`endDate` timestamps), web canonical spec uses recurring weekly windows (`dayOfWeek`/`startTime`/`endTime` strings). Needs resolution.
- **`bookings` collection** — iOS has extra fields (`consultationId`, `tier`, `priceEUR`, `paymentMethod`, `invoiceUrl`) not in web canonical schema. Needs alignment.
- **`time_slot_bookings` pricing** — iOS uses JOD (`originalPriceJOD`/`finalPriceJOD`), web canonical uses USD (`priceUSD`). Two separate systems or unified?
- **CRM naming** — iOS uses `ady_crm` (prefixed), web uses `crm_contacts` (unprefixed). Potential data split.

### iOS Control Parity Request
iOS requests write access to collections currently web-only managed:
- `service_offerings` (CRUD), `bookings` (confirm/cancel/complete), `crm_contacts` (CRUD), `contact_submissions` (reply), `courses` (edit/publish), `video_tutorials` (upload/edit), `project_members` (assign/remove)
- Read access requested for: `membership_plans`, `tap_config`, `audit_log`, `widget_configs`

### Open Questions for Web Team
1. Availability schema — recurring weekly vs concrete slots?
2. CRM collection — consolidate to `crm_contacts`?
3. `service_offerings` schema — should iOS expand to match canonical?
4. Stripe vs Tap — which payment provider?
5. `syncBookingToCalendar` Cloud Function — deployed?
6. `timeline_entries` → `timeline` rename — done on web?
7. Duplicate "First Payment" period deletion — done?
8. `updateEnrollmentCount` Cloud Function — built?

## [1.4.0] - 2026-07-10

### ⚠️ Breaking Changes
- **`timeline_entries` collection renamed to `timeline`** — web updated to match iOS naming. Both platforms now use `timeline`.
- **`site_config/discover_page` schema changed** — photo position now nested as `photoPosition: { scale, offsetX, offsetY }` instead of flat fields.
- **`site_config/google_calendar` schema changed** — renamed `calendarConnected` → `connected`, `calendarConnectedAt` → `connectedAt`.

### Added
- **`site_config` collection** — centralized Firestore configuration for cross-platform real-time sync:
  - `website_sections` document — show/hide website sections (news, projects, courses, membership, tutorials)
  - `discover_page` document — photo zoom/pan position (nested `photoPosition` object)
  - `google_calendar` document — calendar connection status (`connected`, `connectedAt`, `connectedBy`)
  - `roles` document — shared role definitions and permissions
- **`portal_messages` collection** — site-wide messaging system with audience targeting (all/students/clients/candidates/specific), priority levels (normal/important/urgent), bilingual support, read tracking via `readBy` array.
- **`student_crm` collection** — separate student CRM (distinct from `crm_contacts` and `ady_crm`). Includes `mentorshipUnlocked`, `mentorshipUnlockedAt`, `callInvitationId`, `callType`, `callStatus` fields.
- **`call_invitations` collection** — call invitation records for invite-to-call flow. Fields: `callType` (free/discounted), `discountPercent`, `status` (sent/scheduled/completed/expired), `bookedSlotId`, `expiresAt`.
- **`time_slot_config` collection** — tier-based time slot configuration. Fields: `dayOfWeek`, `startHour`, `endHour`, `tier` (premium/regular/discounted/unavailable), `slotDurationMinutes`, `priceJOD`, `maxSeats`, `label`, `labelAr`.
- **`time_slot_bookings` collection** — student slot bookings. Fields: `slotDate` (ISO date), `slotStartHour`, `slotEndHour`, `tier`, `originalPriceJOD`, `finalPriceJOD`, `discountPercent`, `callInvitationId`, `status` (pending/confirmed/completed/cancelled).
- **`latex_templates` collection** — custom LaTeX/CV templates. Fields: `name`, `description`, `source`, `category` (report/invoice/letter/article/custom), `isBuiltIn`.
- **`invite_call` SubmissionResponseType** — new email response type for call invitations with bilingual postcard.
- **`TimeSlotTier` type** — `'premium' | 'regular' | 'discounted' | 'unavailable'`.
- **`syncBookingToCalendar` Cloud Function** — server-side Google Calendar event creation using service account (no client OAuth needed). Both platforms call this for booking calendar sync.
- **Student portal Schedule tab** — seat-booking UI with tier-colored slots, seat availability, pricing display, call invitation banner, auto-discount.
- **Student portal Inbox tab** — reads `portal_messages` filtered by audience + user role/email, marks as read via `readBy` arrayUnion.
- **Website Control panel (iOS)** — admin UI for `site_config` (sections, discover photo, calendar), `portal_messages` compose, `time_slot_config` management. Real-time Firestore sync.

### Changed
- **`portal_messages.readBy`** — now `[String]?` (optional) to handle docs without the field. iOS fixed bug where message ID was incorrectly added instead of student email.
- **`student_memberships`** — web will add `planName` field on create (denormalized from `membership_plans` for easier display).
- **`student_crm.mentorshipUnlocked`** — migrated from localStorage to Firestore. Both platforms read this field instead of local storage.
- **`SubmissionResponseType`** — expanded to include `'invite_call'`.
- **Recurring payment fields confirmed** — both platforms agree on `isRecurring`, `recurrenceFrequency`, `recurrenceEndDate`, `recurrenceParentId`, `label`, `recipientName`, `source` fields.
- **Gemini model** — both platforms now use `gemini-2.5-flash` (web fixed last `gemini-2.0-flash-exp` reference in Transactions.tsx).

### Action Items (Planned)
- **`updateEnrollmentCount` Cloud Function** — will update `course.studentCount` on enrollment create/delete.
- **Web Insights tab** — will build matching iOS Insights view (category breakdown + recurring payments).
- **Delete duplicate "First Payment" period** — web will delete `1764518075230bwtmp8tlg` from `ady_periods`.

### iOS Changes
- **Project consolidation aligned with web team** — 3 projects: ADY, Website Tools, Riad Shannak. Removed `studentTrack` project.
- **ADY removed from main tab bar** — accessed only through Projects hub.
- **Main tab bar cleanup** — removed Submissions (moved to Website Tools), Achievements (removed entirely), Report Generator (broken/removed).
- **New ADY "Insights" tab** — category breakdown, recurring payments management, AI-assisted recurring detection. Uses existing Transaction fields (`isRecurring`, `recurrenceFrequency`, `label`, `recipientName`). No schema changes.
- **Screenshot tool rewritten** — user picks image from photo library, draggable overlays, AI field detection, preset save/load. iOS-only, no web impact.
- **Website Control panel** — 5 sections (Sections, Discover Page, Messages, Time Slots, Calendar) with real-time `site_config` read/write.
- **Student Dashboard (My Track)** — 7 sections: Inbox, Schedule, My Courses, Available Courses, Video Tutorials, Membership, My Application.

### Web Changes
- **Timeline collection renamed** — `timeline_entries` → `timeline` to match iOS.
- **Recurring payment fields added** — Transaction type now includes `isRecurring`, `recurrenceFrequency`, `recurrenceEndDate`, `recurrenceParentId`, `label`, `recipientName`, `source`.
- **Tab count increased** — 28 tabs (added `time-slot-config` and `messaging` to Operations).
- **Student portal tabs** — 6 tabs (added Inbox and Schedule).
- **Invite-to-call flow** — creates `call_invitation` + `student_crm` records, sends bilingual email postcard.
- **Seat-booking system** — tier-based slots with pricing, seat availability, auto-discount for call invitations.

## [1.3.0] - 2026-07-10

### iOS Changes
- **Project consolidation aligned with web team** — 3 projects: ADY, Website Tools, Riad Shannak. Removed `studentTrack` project.
- **ADY removed from main tab bar** — accessed only through Projects hub.
- **Main tab bar cleanup** — removed Submissions (moved to Website Tools), Achievements (removed entirely), Report Generator (broken/removed).
- **New ADY "Insights" tab** — category breakdown, recurring payments management, AI-assisted recurring detection. Uses existing Transaction fields (`isRecurring`, `recurrenceFrequency`, `label`, `recipientName`). No schema changes.
- **Gemini model updated** from `gemini-2.0-flash` to `gemini-2.5-flash` (2.0 shut down June 1, 2026).
- **Screenshot tool rewritten** — user picks image from photo library, draggable overlays, AI field detection, preset save/load. iOS-only, no web impact.

### Action Items for Web Team
- Update Gemini model to `gemini-2.5-flash` if using Gemini on web
- Consider building an Insights view on web portal
- No Firestore schema changes needed

## [1.2.0] - 2026-07-06

### Added
- **`email_responses` collection** — tracks student responses from interactive email buttons. Fields: `id`, `contactId`, `email`, `responseType` (accept/decline/survey), `submittedAt`, `answers[]`.
- **`crm_contacts` updated** — new fields: `responseToken`, `lastResponseType`, `lastResponseAt`.
- **`job_submissions` status expanded** — new values: `interested`, `declined`, `survey_completed`, `enrolled`.
- **`captureEmailResponse` Cloud Function** — public HTTP endpoint for capturing student email responses.
- **`sendSubmissionResponse` Cloud Function** — callable, sends branded HTML emails with interactive response buttons via Resend REST API.
- **`SWIFT_APP_INTEGRATION_GUIDE.md` Section 21** — full iOS integration guide for email response system with Swift code examples.
- **`WEB_RESPONSE_JULY6.md`** — comprehensive response to all open iOS questions and alignment items.

### Changed
- **Web portal tab consolidation: 37 → 20 tabs.** Merged CV templates + invitations into CV Center, video tutorials + assignments into Video Hub, contact messages + careers into Website Manager.
- **SubmissionResponse page** — unified list with funnel filter buttons and search by name/email/role.
- **`isExpense` function** — confirmed `freelance` and `salary` types are treated as expenses (matching iOS default → `.cost` behavior). Added to `Transaction.type` union.
- **`contact_submissions` schema** — added `repliedAt` (timestamp), `replyMessage` (string), `repliedBy` (uid) fields. UI shows "Replied <date>" when set.
- **Duplicate "First Payment" period deleted** — `1764518075230bwtmp8tlg` removed from `ady_periods` in Firestore.

### Added
- **`extractCVData` Cloud Function** — callable, takes `jobSubmissionId`, fetches CV from submission, sends to Gemini for structured extraction, saves `extractedData` + `extractedAt` back on `job_submissions` doc. Returns extracted structured data. iOS can call directly.
- **`sendContactReply` Cloud Function** — callable, takes `contactId` + `replyMessage`, sends branded reply email to contact form submitter, updates `contact_submissions` with `status=replied`, `repliedAt`, `replyMessage`, `repliedBy`.
- **Email deliverability improvements** — plaintext versions added to all email functions, anti-spam headers (`List-Unsubscribe`, `X-Priority`, `X-Mailer`, `Feedback-ID`), consistent `info@the-rs.com` address. See `EMAIL_DELIVERABILITY_GUIDE.md` for DNS setup (SPF/DKIM/DMARC).
