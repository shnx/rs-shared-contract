# Web → iOS: Full Sync Request — Collections, Features, and Project Structure

**From:** Web team (Mohammad Shannak)
**To:** iOS team
**Date:** July 9, 2026
**Priority:** High — please respond within 48 hours

---

## Purpose

We've built several new portal features and fixed structural issues that the iOS team may not be aware of. We need a **bi-directional sync** to make sure both teams are working from the same Firebase collections, the same feature set, and the same project structure.

**Scope:** Portal only. Do not touch the main website except for the Discover page (which is a shared concept that was misunderstood in the deployed code).

---

## 1. What We Need From You (iOS Team)

### 1a. Firebase Collections Audit

Please share the **exact Firebase collection names** your iOS app reads from or writes to, organized by tab/feature. We need this to identify mismatches and duplicates.

For each tab you have, tell us:
- Tab name / ID
- Firebase collection(s) it reads
- Firebase collection(s) it writes
- Whether it's read-only or read-write

### 1b. Features You Built That We Don't Know About

Have you built any portal features, screens, or Cloud Functions that we haven't been informed about? If so, please share:
- Feature name and description
- Collections used
- Cloud Functions called
- Whether it's deployed or in development

### 1c. Features We Built That You May Not Know About

Here's what we've recently built or changed on the web side:

| Feature | Status | Collections | Notes |
|---------|--------|-------------|-------|
| **Discover Page rewrite** | ✅ Deployed | `service_offerings`, `availability`, `bookings` | New "Coffee Time" lead with $3 pricing; free for invited students; reduced from 6 steps to 4 |
| **Coffee Time offering** | ✅ Deployed | `service_offerings` (type: `coffee_time`) | 15-min discovery call with Mohammad Shannak, $3 ("buy me a coffee"), free when invited via submission response |
| **AI Marketing System offering** | ✅ Deployed | `service_offerings` (type: `ai_marketing_system`) | $800 package, 4-6 weeks, company audience |
| **AI Team Empowerment offering** | ✅ Deployed | `service_offerings` (type: `ai_team_empowerment`) | $400 package, 2-3 weeks, company audience |
| **Proposal Generator** | ✅ Deployed | reads `service_offerings` | Portal tab under Operations; generates proposals from templates or offerings; exports as .txt |
| **Offer email links** | ✅ Deployed | — | `sendOfferEmail` Cloud Function now links to `/discover?invited=true` instead of `/book` |
| **Project deduplication** | ✅ Deployed | `projects`, `project_members` | `ProjectContext` now deduplicates by name+type and filters by user membership for non-admins |
| **Tab consolidation** | ✅ Deployed | — | Reduced from 36 tabs to 26 by removing overlaps and tightening project-type filters |
| **Quotation system** | ✅ Deployed | `quotations` | `sendQuotation`, `captureQuotationResponse` Cloud Functions |
| **Booking confirmation** | ✅ Deployed | `bookings` | `sendBookingConfirmation`, `sendBookingReminder` Cloud Functions |

---

## 2. Our Current Firebase Collections (Complete List)

### Core Portal Collections

| Collection Name | Used By | Purpose |
|----------------|---------|---------|
| `projects` | ProjectContext, ProjectSelector, ProjectManagement | Project definitions |
| `project_members` | ProjectContext | User→Project membership mapping |
| `widget_configs` | Dashboard | Per-project widget configuration |
| `audit_log` | Audit system | Action logging |
| `documents` | Documents tab | Portal document management |
| `systemUsers` | Auth | User accounts (Firebase Auth + Firestore) |

### ADY-Specific Collections

| Collection Name | Used By | Purpose |
|----------------|---------|---------|
| `ady_stores` | Store management | Store entities |
| `ady_transactions` | Transactions tab | Financial transactions |
| `ady_periods` | Periods | Accounting periods |
| `ady_installments` | Installments | Payment installments |
| `ady_crm` | CRM tab | CRM leads |
| `comments` | Comments | Transaction/period comments |
| `timeline_entries` | Timeline tab | Project timeline entries |
| `buildings` | Buildings tab | Real estate buildings |

### Student / Education Collections

| Collection Name | Used By | Purpose |
|----------------|---------|---------|
| `courses` | Course Management | Course definitions |
| `lectures` | Course Management | Lecture content |
| `enrollments` | Student Overview | Course enrollments |
| `placement_tests` | Course Management | Placement tests |
| `placement_test_results` | Course Management | Test results |
| `video_tutorials` | Video Hub | Tutorial videos |
| `video_assignments` | Video Hub | Student video assignments |
| `student_invitations` | Student Overview | Invitations sent to students |
| `membership_plans` | Membership Management | Plan definitions |
| `student_memberships` | Student Overview | Active memberships |
| `cv_templates` | CV Preparation | CV templates |
| `prepared_cvs` | CV Preparation | Generated CVs |
| `quizzes` | — | Candidate assessments |
| `career_opportunities` | — | Job board entries |

### Booking / Service Collections

| Collection Name | Used By | Purpose |
|----------------|---------|---------|
| `service_offerings` | Discover page, Service Offerings tab, Proposal Generator | Service packages and pricing |
| `bookings` | Bookings tab, Discover page | Scheduled sessions |
| `availability` | Availability tab, Discover page | Consultant time slots |
| `tap_config` | Tap Config tab | Payment gateway keys |

### Communication Collections

| Collection Name | Used By | Purpose |
|----------------|---------|---------|
| `job_submissions` | Submissions tab, SubmissionResponse | Job applications / CV submissions |
| `contact_submissions` | Contact form | Website contact form entries |
| `email_responses` | Email Inbox | Email receipts / responses |

### Cloud Functions (All `us-central1`)

| Function Name | Purpose |
|---------------|---------|
| `sendCVFeedback` | Email CV feedback to students |
| `sendInvitationEmail` | Email invitations to students |
| `sendSubmissionResponse` | Email response to job applicants |
| `sendOfferEmail` | Email offer (coffee/membership/course) to applicants |
| `sendContactReply` | Reply to contact form submissions |
| `setupAdmin` | Create/repair admin accounts |
| `cleanupSystemUsers` | Remove duplicate user docs |
| `extractCVData` | Gemini AI CV extraction |
| `createTapCheckout` | Create Tap payment session |
| `sendBookingStatus` | Email booking status updates |
| `sendQuotation` | Send quotation to client |
| `sendBookingConfirmation` | Email booking confirmation |
| `sendBookingReminder` | Email booking reminder |

---

## 3. Current Web Portal Tab Structure (26 tabs)

### Overview (6 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `dashboard` | Dashboard | accounting, real_estate, internal |
| `documents` | Documents | all |
| `student-portal` | Student Portal | website, education |
| `investor-portal` | Investor Portal | accounting, real_estate |
| `timeline` | Timeline | accounting |
| `riad-portal` | Riad Portal | real_estate |

### Finance (3 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `finance` | Finance | internal |
| `transactions` | Transactions | accounting |
| `budget` | Budget | accounting |
| `reports` | Reports | accounting, real_estate |

### Student (7 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `submissions` | Submissions | website |
| `student-overview` | Student Management | website, education |
| `course-management` | Courses | website, education |
| `membership-management` | Memberships | website, education |
| `cv-preparation` | CV Center | website, education |
| `video-hub` | Video Hub | website, education |
| `latex-compiler` | LaTeX Compiler | website, education |

### Operations (8 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `crm-contacts` | CRM Contacts | website, education |
| `email-inbox` | Email Inbox | website, education |
| `buildings` | Buildings | real_estate |
| `service-offerings` | Service Offerings | website, consulting |
| `proposal-generator` | Proposal Generator | website, consulting |
| `availability` | Availability | website, consulting |
| `bookings` | Bookings | website, consulting |
| `tap-config` | Tap Config | website |

### Website (1 tab)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `website-manager` | Website Manager | website |

### Administration (6 tabs)
| Tab ID | Label | Project Types |
|--------|-------|---------------|
| `projects` | Projects | all |
| `user-management` | User Management | all |
| `role-management` | Role Management | all |
| `admin-settings` | Admin Settings | internal |
| `agent-log` | Agent Log | internal |
| `testing-lab` | Testing Lab | all |

---

## 4. Critical: Project Structure Decision

**Mohammad's decision:** The portal should have exactly **3 projects** — no more, no less:

| # | Project Name | Type | Purpose |
|---|-------------|------|---------|
| 1 | **ADY** | `accounting` | Accounting, finance, transactions, stores, real estate operations |
| 2 | **Riad Shannak** | `real_estate` | Real estate portfolio, buildings, investor portal |
| 3 | **Website Tools** | `website` | Website management, student portal, courses, CV, bookings, service offerings, proposals, discover page |

### What this means for both teams:

- **Delete duplicate projects** from Firestore — there should be only 3 documents in the `projects` collection
- **No `internal` project type** should appear as a standalone project — internal tabs (finance, admin-settings, agent-log) should be accessible from any project or moved to the `website` project
- **No `education` or `consulting` standalone projects** — education tabs live under the Website Tools project, consulting tabs (service-offerings, availability, bookings, proposal-generator) also live under Website Tools
- **Project membership** should be set up so admin users see all 3, and other users see only the ones relevant to them

### Action items for iOS:
1. Confirm your app shows exactly 3 projects
2. If you have hardcoded project IDs or types, update them to match these 3
3. Remove any references to `internal`, `education`, `consulting`, or `client_interface` as project types — these are now only used for **tab filtering**, not project definitions

---

## 5. Discover Page — Shared Concept

The Discover page (`/discover`) is the only main-website page that both teams should care about. Here's the current state:

### What's deployed (web):
- **Step 1 (Welcome):** Coffee Time hero card ($3 or free if invited) + browse Student/Company packages
- **Step 2 (Packages):** Filtered by path (student/company), shows offerings with outcomes, prices, durations
- **Step 3 (Details):** Full offering details with deliverables, meta info, and "Schedule" CTA
- **Step 4 (Schedule):** Calendar + time slot picker
- **Step 5 (Confirm):** Client info form + payment (Tap) or free booking (if invited)

### URL params we support:
- `?type=coffee_time` — jumps to student packages or coffee schedule
- `?type=coffee_time&invited=true` — free coffee chat, skips payment
- `?type=company_assessment` — jumps to company packages
- `?type=membership` — jumps to student packages
- `?type=course_access` — jumps to student packages

### What we need from iOS:
- Do you have a Discover equivalent? If so, what collections does it read?
- Are you using the same `service_offerings` collection with the same schema?
- Do you support the `invited=true` free booking flow?
- Have you built any booking UI that we should align with?

---

## 6. Summary of Action Items

### iOS team needs to:
1. **Share your complete Firebase collections list** organized by tab
2. **Share any features you built** that we don't know about
3. **Confirm 3-project structure**: ADY, Riad Shannak, Website Tools
4. **Align Discover page** — confirm collection schema and URL param support
5. **Check for duplicate collections** — if you created collections with different names for the same data, flag them so we can consolidate

### Web team has already done:
- ✅ Deduplicated projects in ProjectContext
- ✅ Added user-membership filtering for non-admin users
- ✅ Consolidated tabs from 36 → 26
- ✅ Rewrote Discover page with Coffee Time lead
- ✅ Added AI Marketing System and AI Team Empowerment offerings
- ✅ Built Proposal Generator
- ✅ Updated offer email links to use `/discover?invited=true`
- ✅ Documented all collections and Cloud Functions (above)

---

## 7. Response Format

Please respond in a file named `IOS_SYNC_RESPONSE_JULY9.md` in the `shared/docs/` folder with:

1. Your complete collections list (tab → collections)
2. Any features we don't know about
3. Confirmation of 3-project structure (or objections)
4. Discover page alignment status
5. Any duplicate collections or naming conflicts you've found

**Deadline:** Please respond within 48 hours so we can proceed with cleanup.

---

*This document is shared in `shared/docs/WEB_IOS_SYNC_REQUEST_JULY9.md`*
