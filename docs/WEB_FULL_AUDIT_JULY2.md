# Web → iOS: Full Feature Audit + Alignment Request (July 2, End of Night)

**Date:** July 2, 2026, 11:00pm UTC+2
**From:** Web team
**Priority:** HIGH — please reply with your full feature list

---

## Why We Need This

We've been fixing bugs one by one (wrong collection names, missing tabs, login glitches). Instead of continuing piecemeal, let's do a **full audit** of what both apps have and make sure they match. The user's request: "we should have same options on both iOS and web for many things and actions in each project and they must all work."

---

## Full Firestore Data Audit

We queried every collection. Here's what exists:

### Collections WITH data

| Collection | Docs | Web reads from? | Status |
|---|---|---|---|
| `projects` | 4 | ✅ `projects` | Aligned |
| `ady_transactions` | 66 | ✅ `ady_transactions` (just fixed) | **FIXED** |
| `ady_periods` | 5 | ✅ `ady_periods` (just fixed) | **FIXED** |
| `ady_stores` | 12 | ✅ `ady_stores` (just fixed) | **FIXED** |
| `ady_crm` | 1 | ✅ `ady_crm` | Aligned |
| `timeline_entries` | 67 | ✅ `timeline_entries` | Aligned |
| `job_submissions` | 91 | ✅ `job_submissions` | Aligned |
| `contact_submissions` | 5 | ✅ `contact_submissions` | Aligned |
| `systemUsers` | 25 | ✅ `systemUsers` | Aligned |
| `quizzes` | 17 | ✅ `quizzes` | Aligned |
| `audit_log` | 150 | ✅ `audit_log` | Aligned |
| `widget_configs` | 2 | ✅ `widget_configs` | Aligned |
| `project_members` | 2 | ✅ `project_members` (field mapping fixed) | **FIXED** |

### Collections with 0 docs (empty but structure exists)

| Collection | Docs | Notes |
|---|---|---|
| `ady_installments` | 0 | No installments created yet |
| `buildings` | 0 | Riad Shannak — no buildings added yet |
| `documents` | 0 | No documents uploaded yet |
| `career_opportunities` | 0 | No job postings created yet |
| `subscriptions` | 0 | No subscriptions |
| `comments` | 0 | No comments |
| `stores` (old name) | 0 | Migrated to `ady_stores` |
| `transactions` (old name) | 0 | Migrated to `ady_transactions` |
| `periods` (old name) | 0 | Migrated to `ady_periods` |

### Bugs Fixed This Session

1. **Transactions:** Was reading `transactions` (0 docs) → now reads `ady_transactions` (66 docs)
2. **Periods:** Was reading `periods` (0 docs) → now reads `ady_periods` (5 docs)
3. **Stores:** Was reading `stores` (0 docs) → now reads `ady_stores` (12 docs)
4. **Firestore Timestamp handling:** `ady_transactions.date` is a Firestore Timestamp object `{seconds, nanoseconds}`, not a string. Added `convertDate()` utility that handles Timestamp objects, strings, and Date instances.
5. **Period projectId:** `ady_periods` docs don't have `projectId` field. `getByProject` now returns all periods if none have `projectId`.
6. **Project members field mapping:** Firestore uses `user_id`, `project_id`, `added_at` (snake_case) → now maps to `userId`, `projectId`, `joinedAt` (camelCase).

---

## Web Feature Audit — What Works and What Doesn't

### ADY Project (accounting)

| Tab | What It Does | Data Source | Working? |
|---|---|---|---|
| Launchpad | Quick stats + project overview | `ady_transactions`, `ady_periods` | ✅ Now works |
| Dashboard | Revenue/cost/net widgets | `ady_transactions`, `ady_periods` | ✅ Now works |
| Overview | Project overview | `ady_transactions` | ✅ Now works |
| Budget | Period management + allocation | `ady_periods` | ✅ Now works |
| Transactions | CRUD list of 66 transactions | `ady_transactions` | ✅ Now works |
| Reports | Financial charts + summaries | `ady_transactions` | ✅ Now works |
| Achievements | Timeline of milestones | `timeline_entries` | ✅ Works |
| Documents | Upload/view documents | `documents` (0 docs) | ⚠️ Empty |
| Finance | Widget grid (revenue vs cost) | `ady_transactions` | ✅ Now works |
| Charts | Bar/line/pie charts | `ady_transactions` | ✅ Now works |
| Investor Portal | Investor summary view | `ady_transactions` | ✅ Now works |
| Admin Settings | Project settings | `projects` doc | ✅ Works |

### Main Website Project (website)

| Tab | What It Does | Data Source | Working? |
|---|---|---|---|
| Launchpad | Quick stats | `job_submissions`, `contact_submissions` | ✅ Works |
| Dashboard | Overview widgets | Various | ✅ Works |
| Website | Website content management | `ady_news` (0 docs) | ⚠️ Empty |
| Careers | Job postings CRUD | `career_opportunities` (0 docs) | ⚠️ Empty |
| Submissions | CV/job submissions list | `job_submissions` (91 docs) | ✅ Works |
| Contact Messages | Contact form submissions | `contact_submissions` (5 docs) | ✅ Works |
| CRM | Lead pipeline | `ady_crm` (1 doc) | ✅ Works |
| Video Tutorials | Video upload + management | localStorage only | ⚠️ Not in Firestore |
| Video Assignment | Assignment management | localStorage only | ⚠️ Not in Firestore |
| CV Template Manager | LaTeX CV templates | localStorage only | ⚠️ Not in Firestore |
| CV Preparation | AI extraction + invitations | `job_submissions` | ✅ Works |
| LaTeX Compiler | Compile LaTeX to PDF | N/A (client-side) | ✅ Works |
| Documents | Upload/view documents | `documents` (0 docs) | ⚠️ Empty |
| Admin Settings | Project settings | `projects` doc | ✅ Works |

### Riad Shannak Project (real_estate)

| Tab | What It Does | Data Source | Working? |
|---|---|---|---|
| Launchpad | Quick stats | `buildings` (0 docs) | ⚠️ Empty |
| Dashboard | Building overview | `buildings` (0 docs) | ⚠️ Empty |
| Buildings | Building CRUD + units | `buildings` (0 docs) | ⚠️ Empty |
| Achievements | Timeline of milestones | `timeline_entries` | ✅ Works |
| Documents | Upload/view documents | `documents` (0 docs) | ⚠️ Empty |
| Reports | Building occupancy/rent | `buildings` (0 docs) | ⚠️ Empty |
| Admin Settings | Project settings | `projects` doc | ✅ Works |

### Tools & Admin Project (internal)

| Tab | What It Does | Data Source | Working? |
|---|---|---|---|
| Launchpad | Quick stats | Various | ✅ Works |
| Dashboard | Admin overview | Various | ✅ Works |
| Overview | System overview | Various | ✅ Works |
| User Management | CRUD users | `systemUsers` (25 docs) | ✅ Works |
| Role Management | Role permissions | Hardcoded | ✅ Works |
| Project Management | CRUD projects | `projects` (4 docs) | ✅ Works |
| Projects | Same as above | `projects` | ✅ Works |
| Team | Team members | `project_members` (2 docs) | ✅ Works |
| Testing Lab | API testing | N/A | ✅ Works |
| AI Log | Audit log viewer | `audit_log` (150 docs) | ✅ Works |
| Operations | Operations overview | Various | ✅ Works |
| Settings | Global settings | localStorage | ✅ Works |
| Student Portal | Student management | `systemUsers` | ✅ Works |
| Student Overview | Student detail | `systemUsers` | ✅ Works |
| Student Dashboard | Student self-service | `systemUsers` | ✅ Works |
| Course Management | Course CRUD | localStorage only | ⚠️ Not in Firestore |
| Memberships | Membership plans | localStorage only | ⚠️ Not in Firestore |
| Invitations | CV invitations | `job_submissions` | ✅ Works |
| CV Builder | Student CV builder | localStorage only | ⚠️ Not in Firestore |
| LaTeX Compiler | Compile LaTeX | N/A (client-side) | ✅ Works |
| Documents | Upload/view documents | `documents` (0 docs) | ⚠️ Empty |
| Admin Settings | Project settings | `projects` doc | ✅ Works |

---

## What We Need From iOS

### 1. Your Full Tab List Per Project
Please share the exact tabs you have per project, like we did above. We need to know:
- What tabs exist on iOS for each project
- What features/actions are available in each tab
- What data each tab reads from Firestore

### 2. Feature Parity Gaps
For each feature you have that we don't (or vice versa), let's decide:
- Should both apps have it?
- If yes, who builds it first?
- What collection/schema does it use?

### 3. Data That Lives Only in localStorage (Not in Firestore)
We found several tabs that store data only in browser localStorage — not in Firestore:
- Video Tutorials
- Video Assignment
- CV Template Manager
- Course Management
- Memberships
- CV Builder

**These won't sync between web and iOS.** Should we move them to Firestore? If so, what collection names?

### 4. Empty Collections
These collections exist but have 0 docs:
- `buildings` — Riad Shannak needs building data. Do you have any on iOS?
- `documents` — No documents uploaded. Do you have any?
- `career_opportunities` — No job postings. Do you have any?

### 5. Actions That Should Work on Both
- Create/edit/delete transactions ✅ web can do this
- Create/edit/delete periods ✅ web can do this
- Upload documents ⚠️ web can do this but 0 docs exist
- Create career opportunities ⚠️ web can do this but 0 docs exist
- Manage buildings ⚠️ web can do this but 0 docs exist
- Reply to contact submissions ⚠️ web shows them but no reply function yet
- Create/edit quizzes ✅ web can do this
- Manage users ✅ web can do this

**What can iOS do that we can't? What can web do that iOS can't?**

### 6. Collection Name Confirmation
Please confirm you're reading from these exact collection names:

| Collection | iOS reads? |
|---|---|
| `ady_transactions` | ? |
| `ady_periods` | ? |
| `ady_stores` | ? |
| `ady_crm` | ? |
| `ady_installments` | ? |
| `timeline_entries` | ? |
| `job_submissions` | ? |
| `contact_submissions` | ? |
| `career_opportunities` | ? |
| `buildings` | ? |
| `documents` | ? |
| `systemUsers` | ? |
| `quizzes` | ? |
| `projects` | ? |
| `project_members` | ? |
| `audit_log` | ? |
| `widget_configs` | ? |

---

## Proposed Next Steps

1. **iOS shares full feature list** (tabs + actions per project)
2. **Both teams agree on collection names** (confirm table above)
3. **Move localStorage-only data to Firestore** (video, courses, memberships, CV templates)
4. **Add missing features** on whichever side doesn't have them
5. **Add `repliedAt` to contact_submissions** (iOS requested, we haven't done yet)
6. **Create Cloud Functions** for CV extraction and contact reply (low priority)

Let's align everything so both apps have the same capabilities.
