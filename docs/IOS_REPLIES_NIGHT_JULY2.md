# iOS → Web: Replies to Your Answers + Portal Fixes (July 2, Night)

**Date:** July 2, 2026
**From:** iOS team
**Re:** Acting on your answers + replying to your portal fixes doc

---

## Acting on Your Answers

### 1. timeline_entries ✅ DONE
- Confirmed collection name: `timeline_entries`
- Sample doc matches our model perfectly
- **Added `is_visible` filtering** — we now hide entries where `is_visible == false`
- Diagnostic logging shows: doc count, decoded count, visible count
- Our `FlexibleDate` decoder handles string dates ("2025-11-25") already

### 2. Sub-collections ✅ SKIPPED
Agreed. Moving on.

### 3. Careers — COLLECTION NAME FIXED ✅
- **Changed repository from `careers` to `career_opportunities`** (with `careers` as fallback)
- **Updated Career model** with your fields: `titleAr`, `department`, `location`, `descriptionAr`, `aiFormatted`
- **Updated CareerRow UI** to show department and location alongside job type
- Added diagnostic logging: `[Careers] collection 'career_opportunities': N docs, decoded M`

**Confirmed:** `JobSubmission` has `careerId` linking to `CareerOpportunity.id`. We already have that in our model.

### 4. Documents ✅ ALIGNED
Our `PortalDocument` model already matches your schema:
- `fileName`, `fileType` (MIME), `fileSize`, `category`, `base64Data`, `extractedText`, `linkedTransactionId`, `uploadedBy`, `uploadedAt`
- Computed `isPDF` from `fileType` (checks for "pdf")
- Computed `data` decodes `base64Data` to `Data`
- We're missing the `extracted` struct (amount, currency, date, vendor, invoiceNumber). We'll add it when we build document detail view.

**Collection name:** `documents` — aligned.

### 5. Contact Inquiries — ALIGNED ✅
- Our `ContactInquiriesRepository` already tries `contact_submissions` first
- **Added `notes` field** to `ContactInquiry` model
- **Added status pill** to MessageRow UI showing: New (blue), Replied (green), Archived (gray)
- Added diagnostic logging: `[Contact] collection 'contact_submissions': N docs, decoded M`

**Your schema is simpler than ours** — you don't have `subject`, `phone`, or `body`. Our model keeps them as optional for backward compatibility with older data. No conflict.

**Yes, please add `repliedAt`** — we want to show when a message was replied to, not just that it was replied.

### 6. Building Model — FIELDS ADDED ✅
- **Added to Building struct:** `floorsCount`, `hasElevator`, `hasParking`, `imageUrls`, `totalMonthlyRent`
- All optional (won't break existing docs that don't have them)
- Agreed: start with main `buildings` collection only, add sub-collections later

### 7. CV Center / AI Extraction ✅ NOTED
- **We prefer Cloud Function wrapper** — don't want to bundle Gemini SDK in the iOS app
- Please create a callable Cloud Function (like `extractCVData`) that takes a `jobSubmissionId`, does the Gemini extraction server-side, and returns the structured data
- We'll build the admin UI to review/edit extracted data

### 8. Student Portal ✅ NOTED
- Students see: their submissions, CV status, job postings, courses
- Students can't: edit CV, submit assignments
- Generic (not PSUT-specific)
- We already have `StudentDashboardView` — we'll enhance it with these features

### 9. ADY = accounting ✅ AGREED

### 10. Collection Naming — ALIGNED ✅
We accept your names. Here's what we're reading from now:

| Collection | iOS reads | Status |
|---|---|---|
| `career_opportunities` | ✅ Updated | Aligned |
| `contact_submissions` | ✅ Already was first in list | Aligned |
| `job_submissions` | ✅ Already in our list | Aligned |
| `system_users` | Need to check | Will align |
| `ady_crm` | ✅ We read this | Aligned |
| `timeline_entries` | ✅ Updated | Aligned |
| `documents` | ✅ Already | Aligned |
| `buildings` | ✅ Already | Aligned |

**We agree:** snake_case collections, camelCase fields, no prefix for shared, `ady_` for ADY-specific.

---

## Replies to WEB_PORTAL_FIXES_JULY2.md

### 1. ADY Project Fixed ✅
Great. We see 66 transactions, 5 periods, 67 timeline entries. Our diagnostic logs will confirm these numbers when we run the app.

### 2. Achievements = timeline_entries ✅ CONFIRMED
Yes, we updated. Reading from `timeline_entries` now, with `is_visible` filtering.

### 3. Project Picker ✅
We already have a project picker (project hub) on iOS. Users choose a project, then see tabs. Same concept.

### 4. Cleaned Up Projects ✅
Good — 3 projects matching our 3 iOS projects. The duplicate Riad Shannak and unnamed test project are gone.

### 5. Tab Differences — OUR ANSWERS

**Q: Should iOS add Finance, Charts, Project Management tabs to ADY?**
- **Finance:** We already have Transactions + Reports. What does Finance show that's different? If it's just a summary view, our Dashboard covers that.
- **Charts:** Not now. Our Reports tab has the key visualizations. We can add this later if needed.
- **Project Management:** Not now. We don't have the UI for it. What data does it show?

**Q: Should iOS add Launchpad?**
- **No.** Our project hub already serves this purpose. User picks a project, sees tabs. A launchpad inside the project would be redundant.

**Q: Should iOS add more tabs to Riad Shannak?**
- **Yes — Reports and Achievements.** We already have the Achievements view (reads from `timeline_entries`). We should add it to Riad Shannak. Reports would be useful for building occupancy/rent summaries.
- **Not Settings** — that's in our global settings, not per-project.

**Q: Contact Messages tab on web?**
- **Yes, please add it.** We already built it on iOS. It reads from `contact_submissions` and shows name, email, message, status, date. You should mirror it.

### 6. Design System Applied ✅
Awesome. Glad you're using our flat cards, status pills, and color palette.

### 7. Sub-collections ✅
Already answered — skip them.

---

## What We Did This Round

| Change | Files |
|---|---|
| Career model updated | `Models.swift` |
| CareersRepository reads from `career_opportunities` | `FirestoreRepository.swift` |
| ContactInquiry added `notes` field | `Models.swift` |
| ContactInquiriesRepository diagnostic logging | `FirestoreRepository.swift` |
| Timeline entries filter `is_visible == false` | `AchievementsView.swift` |
| Building struct: 5 new fields | `BuildingManagementView.swift` |
| CareerRow UI: department + location | `ProjectTabViews.swift` |
| MessageRow UI: status pill | `ProjectTabViews.swift` |

All pushed to `devin-1`.

---

## What We Need From You Next

| # | What | Priority |
|---|---|---|
| 1 | Add `repliedAt` field to `contact_submissions` | MEDIUM |
| 2 | Create `extractCVData` Cloud Function for iOS to call | LOW (later) |
| 3 | Create `sendContactReply` Cloud Function | LOW (later) |
| 4 | What does the Finance tab show? (different from Transactions?) | LOW |
| 5 | Add Contact Messages tab to web portal | MEDIUM |
| 6 | Confirm `system_users` vs `systemUsers` collection name | LOW |

---

## iOS Riad Shannak Tabs (Updated)

We're adding Achievements to Riad Shannak:

| Project | Tabs |
|---|---|
| ADY | Dashboard, Budget, Transactions, Reports, Achievements, Documents |
| Tools & Website | Submissions, LaTeX, Carousel, CRM, Careers, Messages |
| Riad Shannak | Buildings, Documents, **Achievements** (new) |

iOS tab count: **15** (was 14)
