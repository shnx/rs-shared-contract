# Web → iOS: Comprehensive Response to All Open Items (July 6)

**Date:** July 6, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Responding to all open iOS requests, confirming alignments, and sharing latest web updates

---

## Status: We're Caught Up

We've been heads-down building the student funnel, email response system, and tab consolidation. Here's a full response to everything you've asked for.

---

## 1. Replies to IOS_QUESTIONS_JULY2.md (10 Questions)

### Q1: timeline_entries ✅ CONFIRMED
- Collection name is exactly `timeline_entries` (snake_case)
- `date` is a string ("2025-11-25"), not a Firestore Timestamp
- `is_visible` filtering: YES, hide entries where `is_visible == false`
- Sample doc (as you already received in WEB_REPLY_TO_IOS_QUESTIONS.md)

### Q2: Sub-collections ✅ SKIPPED — agreed

### Q3: Careers collection schema
Our `CareerOpportunity` type:
```
id, title, titleAr, description, descriptionAr, department, location,
jobType ('full-time' | 'part-time' | 'internship' | 'contract'),
requirements (string[]), salaryRange, isActive, isRemote,
aiFormatted, createdAt, updatedAt
```
Collection name: `career_opportunities` (you already updated — ✅ aligned)
`JobSubmission` has `careerId` linking to `CareerOpportunity.id` — ✅ you already have this.

### Q4: Documents schema ✅ ALIGNED
Our `PortalDocument` fields: `fileName, fileType (MIME), fileSize, category, base64Data, extractedText, linkedTransactionId, uploadedBy, uploadedAt, projectId`
The `extracted` struct (amount, currency, date, vendor, invoiceNumber) is set during PDF upload via OCR. You can ignore it for now.

### Q5: Contact inquiries
Our `ContactSubmission` fields: `name, email, message, status ('new' | 'read' | 'replied' | 'archived'), submittedAt, notes`
- We DON'T have `repliedAt` yet — **we'll add it this week**
- We DON'T have `priority` — not needed for now
- Collection name: `contact_submissions` — ✅ you already aligned

### Q6: Building model ✅ AGREED
Your proposed schema with `floorsCount, hasElevator, hasParking, imageUrls, totalMonthlyRent` looks great. We'll start with the main `buildings` collection only. Sub-collections (units, tenants, rentPayments, maintenanceTickets) later.

### Q7: CV Center / AI extraction
- We use **Google Gemini** (gemini-1.5-flash) via Cloud Function
- The extraction prompt sends the CV as base64 + asks for structured JSON
- Extracted data is stored on the `jobSubmissions` document in an `extractedData` field
- **We'll create a callable `extractCVData` Cloud Function** that takes a `jobSubmissionId` and returns structured data — you can call it from iOS
- Timeline: This week

### Q8: Student portal scope
- Students see: their submissions, CV status, prepared CVs, courses, video tutorials
- Students can: view content, build their own CV, download prepared CVs
- Students can't: edit admin-prepared CVs, submit assignments (yet)
- Generic, not PSUT-specific
- We already have `StudentDashboard` and `StudentCVBuilder` pages

### Q9: ADY = accounting ✅ AGREED

### Q10: Collection naming convention ✅ AGREED
- No prefix for shared collections
- `ady_` for ADY-specific
- snake_case collections, camelCase fields

---

## 2. Replies to IOS_URGENT_ANSWERS.md (5 Calculation Issues)

### Issue 1: Duplicate "First Payment" period
**Action needed:** We need to delete `1764518075230bwtmp8tlg` from Firestore. We'll do this this week. Thanks for the diagnosis — keeping the UUID-based one (`CBE91C2E-...`).

### Issue 2: `freelance` and `salary` transaction types
**We'll update `isExpense` to match your approach:**
```javascript
function isExpense(tx) {
  if (tx.isExpense != null) return tx.isExpense;
  const t = (tx.type || '').toLowerCase();
  if (t === 'revenue' || t === 'payment' || t === 'check') return false;
  return true; // everything else is expense (matches iOS default → .cost)
}
```
This is the right approach — anything not explicitly income is expense.

### Issue 3: EUR currency ✅ NO CHANGE NEEDED
Both platforms treat EUR as JOD (1:1). We'll add proper EUR support later together.

### Issue 4: Date range vs periodId ✅ NO CHANGE NEEDED
Our OR logic (`dateInRange || periodId match`) is correct and matches iOS.

### Issue 5: Exact numbers
After fixing #1 (delete duplicate period) and #2 (freelance/salary as expense), our numbers should match:
- Total Spent: ~5,567.45 JOD
- In Cash: ~-167.45 JOD (negative — show red)

---

## 3. Replies to IOS_REPLIES_NIGHT_JULY2.md (6 Items Needed From Us)

| # | What You Need | Status |
|---|---|---|
| 1 | Add `repliedAt` to `contact_submissions` | **TODO this week** |
| 2 | Create `extractCVData` Cloud Function | **TODO this week** |
| 3 | Create `sendContactReply` Cloud Function | **TODO this week** |
| 4 | What does Finance tab show? | See below |
| 5 | Add Contact Messages tab to web portal | ✅ **DONE** — merged into Website Manager tab |
| 6 | Confirm `system_users` vs `systemUsers` | Collection is `systemUsers` (camelCase, not snake_case — exception because it predates the convention) |

### What the Finance tab shows (your Q4):
The Finance tab is a **widget-grid overview** with summary cards:
- Total Income, Total Spent, Net Balance, Transaction Count
- Period summaries (allocated vs spent vs remaining)
- Funding progress (PSUT grant: received / total / % )
- Overspend warning

It's essentially a dashboard for financial health. Your Budget tab covers the same data. **No need to add Finance to iOS** — your Budget + Dashboard combo is sufficient.

---

## 4. What We've Built Since You Last Heard From Us (July 4-6)

### A. Interactive Email Response System
- Emails sent to students now include interactive buttons: **Interested**, **Not Now**, **Survey**
- Public Cloud Function `captureEmailResponse` captures responses
- New `email_responses` Firestore collection
- `crm_contacts` updated with `responseToken`, `lastResponseType`, `lastResponseAt`
- `job_submissions` status expanded: `interested`, `declined`, `survey_completed`, `enrolled`
- Public response page at `/email-response?token=xxx&action=accept`

**iOS action needed:** See Section 21 of `SWIFT_APP_INTEGRATION_GUIDE.md` for full details and Swift code examples.

### B. Tab Consolidation (37 → 20 tabs)
The portal was too crowded. We merged:
- CV Templates + Invitations → into CV Center (already had sub-tabs)
- Video Tutorials + Video Assignments → **Video Hub** (new, with sub-tabs)
- Contact Messages + Careers → **Website Manager** (new, with sub-tabs)
- Email Responses → sub-tab inside Student Overview
- Removed placeholder tabs (operations, website)

**Current web tab structure:**

| Category | Tabs |
|---|---|
| Overview | Launchpad, Dashboard, Documents, Student Dashboard, Student Portal, Investor Portal, Timeline, Riad Portal |
| Finance | Finance, Transactions, Budget, Reports |
| Student | Submissions, Submission Response, CRM Contacts, Student Overview, Course Mgmt, Membership Mgmt, CV Center, CV Builder, Video Hub |
| Operations | LaTeX Compiler, Email Inbox, Buildings, CRM |
| Website | Website Manager (messages + careers) |
| Admin | Projects, User Mgmt, Role Mgmt, Settings, Agent Log, Testing Lab |

### C. Submission Response UX Improvements
- Unified list with funnel filter buttons: All / New / Responded / Interested / Survey Done / Declined
- Search by name, email, or role
- Stage badges on each submission card
- CRM Contacts page shows response status, last response date, and email response history in detail modal

### D. Swift Integration Guide Updated
`SWIFT_APP_INTEGRATION_GUIDE.md` now includes Section 21 with:
- `email_responses` collection schema
- Updated `crm_contacts` and `job_submissions` fields
- `captureEmailResponse` Cloud Function endpoint + Swift code examples
- Student funnel pipeline diagram
- Updated portal tab structure

---

## 5. Open Action Items (Both Teams)

### Web team will do this week:
1. Delete duplicate "First Payment" period from Firestore
2. Update `isExpense` to treat freelance/salary as expenses (match iOS)
3. Add `repliedAt` field to `contact_submissions`
4. Create `extractCVData` callable Cloud Function
5. Create `sendContactReply` Cloud Function
6. Reply to GitHub issue #1 on rs-shared-contract

### iOS team — please confirm:
1. Have you received the `SWIFT_APP_INTEGRATION_GUIDE.md` update (Section 21)?
2. Can you add `email_responses` collection to your iOS models?
3. Can you update `JobSubmission` status enum with: `interested`, `declined`, `survey_completed`, `enrolled`?
4. The `systemUsers` collection name is camelCase (exception to snake_case convention) — is this OK on your side?

---

## 6. GitHub Issue #1 Response

We'll reply on https://github.com/shnx/rs-shared-contract/issues/1 with:
- ✅ Data contract read and understood
- ✅ Shared link feature already implemented on web (client-side rendering from `sharedViews` doc)
- ✅ Permissions schema expanded to 14 fields (including `canManageSubscriptions`, `canGenerateReports`, `canComment`, `canManageProjects`)
- ✅ `manager` role added
- ⚠️ Collection name exception: `systemUsers` is camelCase (predates convention)
- 📋 New collections added since contract v1.1: `email_responses`, `crm_contacts` (updated fields)
- 📋 New Cloud Functions: `captureEmailResponse` (HTTP), `sendSubmissionResponse` (callable)

---

## TL;DR

We're aligned on almost everything. The main open items are:
1. **Web:** Fix 2 calculation bugs (duplicate period + freelance/salary classification)
2. **Web:** Create 2 Cloud Functions (`extractCVData`, `sendContactReply`)
3. **Web:** Add `repliedAt` to contact submissions
4. **iOS:** Add `email_responses` collection + new `job_submissions` statuses
5. **Both:** Verify numbers match after fixes

We'll push the calculation fixes and Cloud Functions this week.

---

*Reply by adding a new `.md` file to `shared/docs/` or comment on GitHub issue #1.*
