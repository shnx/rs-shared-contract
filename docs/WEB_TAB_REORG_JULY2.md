# Web → iOS: Tab Reorganization Complete + Question (July 2, Late Night)

**Date:** July 2, 2026, 10:50pm UTC+2
**From:** Web team

---

## What We Did

### 1. Fixed Login Glitches
- **Race condition fixed:** Legacy admin login now sets localStorage BEFORE signing out Firebase Auth, preventing `onAuthStateChanged` from overriding the session
- **No more auto-student creation:** If a Firebase Auth user has no profile in `systemUsers`, we sign them out instead of auto-creating a student/candidate profile
- **Loading spinner:** Project selector shows a spinner while projects load from Firestore (no more flash of "no projects")

### 2. Strict Per-Project Tabs — No More Everything Everywhere

Every tab was listed for every project type. Now each project has only its essential tabs:

| Project | Type | Tabs |
|---|---|---|
| **ADY** | accounting | Launchpad, Dashboard, Overview, Budget, Transactions, Reports, Achievements, Documents, Finance, Charts, Investor Portal, Admin Settings (12) |
| **Main Website** | website | Launchpad, Dashboard, Website, Careers, Submissions, Contact Messages, CRM, Video Tutorials, Video Assignment, CV Template Manager, CV Preparation, LaTeX Compiler, Admin Settings, Documents (14) |
| **Riad Shannak** | real_estate | Launchpad, Dashboard, Buildings, Achievements, Documents, Reports, Admin Settings (7) |
| **Tools & Admin** (NEW) | internal | Launchpad, Dashboard, Overview, LaTeX Compiler, Testing Lab, AI Log, User Management, Role Management, Project Management, Projects, Team, Operations, Settings, Student Portal, Student Overview, Student Dashboard, Course Management, Memberships, Invitations, CV Builder, Admin Settings, Documents (22) |

### 3. New Project: Tools & Admin
Created a 4th project (`type: internal`) for all the utility/admin/student tabs that don't belong to a specific business project. This includes user management, role management, testing lab, student tools, CV builder, course management, etc.

### 4. Final Firestore Projects (4 total)
```
ADY (accounting) — 12 tabs
Main Website (website) — 14 tabs
Riad Shannak (real_estate) — 7 tabs
Tools & Admin (internal) — 22 tabs
```

---

## Question for iOS

**What tabs do you currently have per project?** We need to align. You mentioned 15 tabs across 3 projects:

| iOS Project | Tabs (from your last doc) |
|---|---|
| ADY | Dashboard, Budget, Transactions, Reports, Achievements, Documents (6) |
| Tools & Website | Submissions, LaTeX, Carousel, CRM, Careers, Messages (6) |
| Riad Shannak | Buildings, Documents, Achievements (3) |

**Questions:**
1. Is this still your current tab list, or have you added/removed any?
2. Do you want a "Tools & Admin" project on iOS too? We created one on web for all the admin/utility tabs (user management, roles, testing lab, student tools, etc.)
3. Our web has 4 projects now. You have 3. Should we keep 4, or merge Tools & Admin into one of the existing projects?
4. We added `repliedAt` to your todo — do you still need it? We haven't added it to the schema yet. Let us know if it's blocking.

---

## Answers to Your Previous Questions

**Finance tab vs Transactions:** Finance is a widget-grid summary view (revenue vs costs chart, budget vs actual, cash flow). Transactions is the detailed CRUD list. Finance = overview, Transactions = data entry.

**`system_users` vs `systemUsers`:** We use `systemUsers` (camelCase) in our code. But per our naming convention (snake_case collections), it should be `system_users`. We'll rename it. Currently the collection is empty (0 docs) — we use legacy admin. So no data loss.

---

All deployed. Please pull and share your current tab list so we can align.
