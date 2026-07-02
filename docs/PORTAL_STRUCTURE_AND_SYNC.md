# Web → iOS: Portal Structure, Testing Lab, User Management Fixes + Sync Plan

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Current portal structure, new testing lab, user management fixes, and proposal for keeping in sync

---

## Current Portal Structure (Web)

The web portal is organized into 6 categories with the following tabs:

### Overview
| Tab ID | Label | Roles | Description |
|--------|-------|-------|-------------|
| `launchpad` | Launchpad | All | Visual home page with quick links |
| `dashboard` | Dashboard | admin, manager, advisor | Project overview with widgets |
| `documents` | Documents | admin, manager, advisor | Document management |
| `student-dashboard` | My Dashboard | student, candidate | Student's personal dashboard |
| `investor-portal` | Investor Portal | client | Investor/client overview |

### Finance
| Tab ID | Label | Roles | Description |
|--------|-------|-------|-------------|
| `finance` | Finance | admin, manager, advisor | Financial overview with widgets |
| `transactions` | Transactions | admin, manager, advisor | Transaction management |
| `budget` | Budget | admin, manager, advisor | Budget management |
| `reports` | Reports | admin, manager, advisor | Reports and charts |

### Student
| Tab ID | Label | Roles | Description |
|--------|-------|-------|-------------|
| `cv-preparation` | CV Center | admin | CV templates, preparation, invitations |
| `cv-builder` | CV Builder | student | Student-facing CV generation tool |
| `student-overview` | Student Management | admin | Manage students, view portal |
| `course-management` | Courses | admin | Course and lecture management |
| `membership-management` | Memberships | admin | Membership plans management |
| `video-assignment` | Video Assignments | admin, student | Video assignment management |

### Operations
| Tab ID | Label | Roles | Description |
|--------|-------|-------|-------------|
| `operations` | Operations | admin, manager | Operations placeholder |
| `latex-compiler` | LaTeX Compiler | admin, student | Compile LaTeX documents |

### Website
| Tab ID | Label | Roles | Description |
|--------|-------|-------|-------------|
| `website` | Website Settings | admin | Website configuration |
| `submissions` | Submissions | admin | Job application submissions |
| `contact_messages` | Messages | admin | Contact form messages |
| `careers-management` | Careers | admin | Career posting management |
| `video-management` | Video Tutorials | admin | Video tutorial management |

### Administration
| Tab ID | Label | Roles | Description |
|--------|-------|-------|-------------|
| `projects` | Projects | admin | Project management |
| `user-management` | User Management | admin | User CRUD, impersonation, candidates |
| `role-management` | Roles | admin | Role and permission management |
| `admin-settings` | Settings | admin | System settings |
| `agent-log` | Agent Log | admin | AI agent action log |
| `testing-lab` | Testing Lab | admin | **NEW** — test emails, fake users, flows |

---

## New: Testing Lab

We've added a **Testing Lab** tab (`testing-lab`) in the Administration section. It allows admins to:

1. **Send test emails** — CV feedback and invitation emails to any address
2. **Create fake students** — individual or bulk (5 pre-defined test accounts)
3. **Create test invitations** — in Firestore, then send emails
4. **View recent invitations** — with status and direct links to the registration page
5. **Clean up test data** — delete fake user profiles from Firestore

All fake users get password `TestPass123!` and emails at `*.test.rs@example.com`.

### How to test the full flow
1. Create a fake student (Testing Lab → Fake Users)
2. Send a CV feedback email (Testing Lab → Email Testing)
3. Open the registration link from the email in a new tab
4. Complete onboarding — set credentials, see feedback, open CV Builder
5. Log in as the student (User Management → impersonate, or direct login)
6. Clean up when done (Testing Lab → Cleanup)

---

## User Management Fixes

### Fixed: `registerWithPassword` signing out admin
- **Issue:** `AuthContext.registerWithPassword()` used `firebaseAuth.signUp()` (primary auth instance), which would sign out the current admin session when called
- **Fix:** Switched to `firebaseAuth.createUserAsAdmin()` (secondary auth instance), which creates the user without affecting the admin's session

### Current user management flow (all working)
- **Create user:** `createUserAsAdmin()` via secondary auth → `setDoc()` in Firestore with UID
- **Edit user:** `updateDoc()` on Firestore `systemUsers` document
- **Delete user:** `deleteDoc()` on Firestore (Firebase Auth account needs manual deletion in console)
- **Impersonate:** Client-side swap — stores admin in `localStorage('portal_impersonating')`, uses ref guard to prevent `onAuthStateChanged` from overriding
- **Bulk migration:** Creates Firebase Auth accounts for passwordless users, sends password reset emails
- **Set credentials:** Users without `hasPassword: true` are redirected to `SetCredentialsPage`

---

## Proposal: Keeping Web and iOS in Sync

### Problem
The portal is getting complex and both platforms need to stay aligned. Currently we use the `shared/` submodule for data contracts, but there's no structured way to coordinate feature development.

### Proposal: Shared feature tracking

1. **Shared roadmap doc** — Add `shared/docs/ROADMAP.md` listing all planned features with:
   - Feature name and description
   - Which platform(s) are responsible
   - Status: `planned`, `in-progress`, `done`
   - Dependencies between platforms

2. **Tab ID registry** — Add `shared/schemas/portal-tabs.json` listing all tab IDs, labels, and roles so both platforms show the same navigation structure

3. **Bi-weekly sync** — Every 2 weeks, both teams update `shared/docs/STATUS_<date>.md` with:
   - What was completed
   - What's in progress
   - Blockers or questions

4. **Breaking change protocol** — When changing a Firestore schema or adding a required field:
   - Update `shared/schemas/*.json` first
   - Bump `CHANGELOG.md` with breaking change tag
   - Both teams pull before deploying
   - Add backward compatibility for 1 release cycle

### What we need from iOS team
1. **Confirm the tab structure** above matches your iOS app structure (or send us your current structure)
2. **Let us know which features you're working on** so we can add them to the roadmap
3. **Confirm the testing approach** — do you have a similar testing setup, or should we share test user accounts?
4. **Portal reorganization** — we know the portal is confusing. We want to reorganize but don't want to break iOS. Can we agree on a transition plan?

---

## Current Firestore Collections (Full List)

| Collection | Purpose | Public Read |
|-----------|---------|-------------|
| `systemUsers` | User profiles (UID as doc ID) | No |
| `studentInvitations` | CV feedback invitations | No |
| `preparedCVs` | Admin-prepared and student-built CVs | No |
| `cvTemplates` | LaTeX CV templates | No |
| `courses` | Course definitions | No |
| `lectures` | Course lectures | No |
| `enrollments` | Student course enrollments | No |
| `placement_tests` | Course placement tests | No |
| `placement_test_results` | Test results | No |
| `video_tutorials` | Video tutorial catalog | No |
| `video_assignments` | Student video assignments | No |
| `membership_plans` | Membership tier definitions | No |
| `student_memberships` | Student membership records | No |
| `submissions` | Job applications | No |
| `sharedViews` | Shareable budget dashboard tokens | **Yes** |
| `venture_reports` | Public venture report pages | **Yes** |
| `timeline` | Achievement photos for reports | **Yes** |
| `transactions` / `ady_transactions` | Financial transactions | No |
| `periods` / `ady_periods` | Budget periods | No |
| `documents` / `ady_documents` | Project documents | No |
| `projects` | Project definitions | No |

---

*Reply by adding a new `.md` file to `shared/docs/` — we'll pull and read it.*
