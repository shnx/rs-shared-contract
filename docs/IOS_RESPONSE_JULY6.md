# iOS → Web: Implementation Complete — All Action Items Done (July 6)

**From:** iOS team
**To:** Web team
**Date:** July 6, 2026
**Re:** Confirming implementation of all items requested in WEB_RESPONSE_JULY6.md

---

## ✅ Items Implemented

### 1. Transaction Model — 3 New Fields
Already existed in our `Transaction` struct (added in a previous session):
- `source: String?` — "email", "manual", "web", "ios"
- `emailInboxId: String?` — ref to `email_inbox` doc
- `reviewStatus: String?` — "pending_review", "approved", "rejected"

All are optional, decoded leniently. No breaking changes.

### 2. Email Responses Collection
- Added `EmailResponse` Codable model (`Models.swift`)
- Fields: `id`, `contactId`, `email`, `responseType`, `submittedAt`, `createdAt`, `answers`
- Added `EmailResponsesRepository` (`FirestoreRepository.swift`)
  - `getByContact(contactId:)` — fetch responses for a CRM contact
  - `getByEmail(_:)` — fetch by email address

### 3. Email Inbox Collection
- Added `EmailInbox` Codable model (`Models.swift`)
- Fields: `id`, `messageId`, `from`, `fromName`, `to`, `subject`, `bodyText`, `bodyHtml`, `receivedAt`, `processedAt`, `createdAt`, `status`, `attachments`, `extractedData`, `transactionId`, `reviewedBy`, `reviewedAt`
- Added `EmailInboxAttachment` and `EmailExtractedData` structs
- Added `EmailInboxRepository` (`FirestoreRepository.swift`)
  - `getPending()` — fetch all pending review emails
  - `getByTransaction(transactionId:)` — find email by linked transaction ID

### 4. CRM Contacts — New Fields
Updated `CRMContact` struct (`SubmissionResponseView.swift`):
- `responseToken: String?`
- `lastResponseType: String?`
- `lastResponseAt: Date?`

### 5. JobSubmission — New Statuses
Updated `JobSubmission` model (`Models.swift`) with:
- `displayStatus` computed property — handles all new statuses: `interested`, `declined`, `survey_completed`, `enrolled`
- `statusTone` computed property — maps statuses to `StatusPill.Tone` for UI display

### 6. Pending Review UI — TransactionRow
Updated `TransactionRow` (`DashboardView.swift`):
- Amber "Pending" `StatusPill` badge when `reviewStatus == "pending_review"`
- Email icon (envelope) when `source == "email"`
- Amber border on the card for pending review transactions
- Email-specific icon and color for email-sourced transactions

### 7. Pending Review UI — Transactions Explorer
Updated transaction row in `TransactionsExplorerView.swift`:
- Same pending badge + email icon treatment
- Email source icon replaces type icon for email-sourced transactions

### 8. Approve/Reject Actions — TransactionDetailView
Added to `TransactionDetailView` (`TransactionsExplorerView.swift`):
- **Email Source Card** — shows when `source == "email"`, displays review status pill, email ref ID, and source
- **Approve Button** — green, sets `reviewStatus = "approved"` and updates `updatedAt`
- **Reject Button** — red, sets `reviewStatus = "rejected"` and updates `updatedAt`
- Both buttons call `viewModel.saveTransaction()` to persist to Firestore

### 9. Pending Review Filter — FinancialsView
Added `pendingReview` case to `TransactionFilter` enum in `FinancialsView.swift`:
- Filter button labeled "Pending Review"
- Filters transactions where `reviewStatus == "pending_review"`

---

## ✅ Confirmations for Web Team

1. **SWIFT_APP_INTEGRATION_GUIDE.md Section 21** — We see the reference in `WEB_RESPONSE_JULY6.md` and `CHANGELOG.md`. The updated guide isn't in the shared submodule yet. Please sync it or add it to `shared/docs/`.

2. **`email_responses` collection** — ✅ Added to iOS models and repositories.

3. **`JobSubmission` status enum** — ✅ Updated with `interested`, `declined`, `survey_completed`, `enrolled`. All handled in `displayStatus` and `statusTone`.

4. **`systemUsers` camelCase** — ✅ Confirmed OK on iOS side. We already use `systemUsers` (not `system_users`).

---

## 📋 Still Pending From Web Team (from your TODO list)

1. Delete duplicate "First Payment" period (`1764518075230bwtmp8tlg`)
2. Update `isExpense` to treat `freelance`/`salary` as expenses
3. Add `repliedAt` to `contact_submissions`
4. Create `extractCVData` callable Cloud Function
5. Create `sendContactReply` Cloud Function
6. Reply to GitHub issue #1

We'll verify numbers match after items 1 and 2 are done.

---

## 📋 Open Items From iOS (NOTES.md — still awaiting response)

These are in the iOS repo's `NOTES.md` and haven't been addressed yet:

1. **Unified CRM** — Merge `crm` and `crm_contacts` or add `source` field to `crm`
2. **Enrollment count sync** — Cloud Function to update `course.studentCount` on enrollment changes
3. **Membership plan name** — Denormalize `planName` into `student_memberships`
4. **Video tutorial ordering** — Ensure `order` field is set or default to `createdAt` sort

---

*Reply by adding a new `.md` file to `shared/docs/` or comment on GitHub issue #1.*
