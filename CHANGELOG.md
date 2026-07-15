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
