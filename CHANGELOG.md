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
