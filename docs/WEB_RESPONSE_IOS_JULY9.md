# Web → iOS: Response to Full Sync Spec

**From:** Web team | **Date:** July 9, 2026

## Auth ✅ Already Built
Username/email, Apple, Google, legacy migration, password reset — all done. Need env vars: `VITE_GOOGLE_CLIENT_ID`, `VITE_APPLE_CLIENT_ID`, `VITE_APPLE_REDIRECT_URI`.
**Q:** Roles `admin|manager|investor` vs our `admin|manager|advisor|client|student|candidate` — map `investor`→`client`?

## Pricing ✅ Accepted
€15 First Coffee, EUR students, USD companies, 6 tiers. Will update seed data + Discover page.

## Ecosystem ✅ Accepted
First Coffee → assessment → nurture → mentoring → community. No aggressive upsell.

## 3 Projects
ADY (accounting), Riad Shannak (real_estate), Website Tools (website). Confirm you match.

## Collections to Reconcile
- `service_offerings` → migrate to `consultations` (map our types to your tiers)
- `ady_crm` vs `crm` — same or different? Please clarify
- `email_responses` vs `email_inbox` — different features, keep both

## Collections We'll Add
`milestones`, `crm_contacts`, `builder_portfolios`, `talent_matches`, `monthly_content`, `project_challenges`, `periodSummaries`, `sharedViews`, `ady_meetings`, `ady_workSessions`, `ady_employeePermissions`, `ady_pdf_meta`

## Tabs — Need Your List
We have 26 tabs. Please share your exact tab list with collections per tab. We'll reconcile to one structure.

## Cloud Functions — Will Build
`sendWelcomeToCommunity`, `sendMonthlyCommunityDigest`, `calculateLeadScores`. We already have 13 deployed.

## Booking Schema — Will Align
Will add `consultationId`, `tier`, `paymentMethod`, `invoiceUrl`, `priceEUR` to `bookings` collection.

## Immediate Fixes Done
- Documents.tsx Invalid Date crash fixed (safeDate wrapper)
- Project dedup by name+type
- Server connection errors handled gracefully
