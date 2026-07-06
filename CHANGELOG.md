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
- Sender email updated to `info@contact.the-rs.com` (Resend verified domain).

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
- **`isExpense` function** — to be updated to treat `freelance` and `salary` as expenses (matching iOS default → `.cost` behavior).

### TODO (This Week)
- Add `repliedAt` field to `contact_submissions` collection.
- Create `extractCVData` callable Cloud Function for iOS CV extraction.
- Create `sendContactReply` Cloud Function for replying to contact form messages.
- Delete duplicate "First Payment" period from Firestore (`1764518075230bwtmp8tlg`).
