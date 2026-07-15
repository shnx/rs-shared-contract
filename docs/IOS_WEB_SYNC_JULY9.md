# iOS → Web Team Sync — July 9

## 1. Settings Reorganization (Done)

The iOS Settings tab has been cleaned up:

**Kept:**
- User Profile card (name, email, role badge)
- Account section (Profile, Notifications, Security)
- Administration section (Manage Users, Employee Management) — admin only
- Switch View (temporarily view app as another role) — admin only
- About (version, help, terms, privacy)
- Sign Out

**Removed:**
- Customize Tabs — not needed, role-based tabs are sufficient
- Theme Customization — was a stub with non-functional buttons
- Setup & Diagnostics — was a one-time debug tool for provisioning a single user (Riad)

**User Management sync:**
- Users created on iOS write to Firestore `systemUsers` collection — web portal sees them immediately
- Role changes on iOS update `systemUsers.role` in Firestore — web portal picks them up
- Limitation: Creating a user from iOS client SDK signs the admin out temporarily (Firebase Auth requirement). Web portal should prefer creating users via admin SDK.
- Both platforms read/write the same `systemUsers` collection — no separate sync needed.

---

## 2. Tab-by-Tab Data Model & Presentation Sync

The iOS app has these tabs. The web portal should mirror each one with the same Firestore collections and data shapes. Presentation adapts to screen type (mobile tab bar vs web sidebar/pages).

### Tab: Dashboard
- **Firestore:** `projects`, `transactions`, `periods`, `periodSummaries`
- **iOS shows:** KPI cards (total fund, spent, remaining), recent transactions, spending breakdown by category
- **Web should show:** Same KPIs in a wider grid, plus charts (bar/line) for spending over time
- **Key fields:** `amount`, `currency`, `type` (cost/revenue), `fundingSource`, `periodId`

### Tab: Projects
- **Firestore:** `projects` collection
- **iOS shows:** List of projects with status, tapping opens ProjectWorkspaceView
- **Web should show:** Project cards/grid with same status badges, clicking opens project detail page
- **Key fields:** `title`, `description`, `status`, `teamMembers`, `budget`

### Tab: ADY (Project Workspace)
- **Firestore:** `transactions` (where `projectId == ADY`), `periods`, `documents`, `periodSummaries`, `ady_pdf_meta`
- **iOS shows:** 5 sections — Overview, Periods, Transactions, Payments, Reports
- **Web should show:** Same 5 sections as tabs or pages within project detail
- **Payments section (NEW):** Organizes transactions under user-defined labels (Salary, AI Development, Purchases, Design Cost, etc.). Supports recurring payment flags and recipient names. Bulk label assignment for old transactions with messy descriptions.
- **Key new fields on `transactions`:**
  - `label` (string, nullable) — user-defined grouping label
  - `recipientName` (string, nullable) — person or company receiving payment
  - `isRecurring` (bool, nullable) — marks recurring payments
  - `recurrenceFrequency` (string: "monthly", "biweekly", "weekly")
  - `recurrenceEndDate` (date, nullable)
  - `recurrenceParentId` (string, nullable) — links recurring instances

### Tab: Submissions
- **Firestore:** `crm_contacts` (submissions are stored as CRM contacts with `submissionId`)
- **iOS shows:** List of submissions with status badges, review actions, evidence viewer
- **Web should show:** Submissions table with filters, bulk approve/reject, same status workflow
- **Key fields:** `status` (pending_review, approved, rejected), `name`, `email`, `role`, `proposedCourses`, `proposedServices`

### Tab: CRM
- **Firestore:** `crm_contacts`
- **iOS shows:** Contact list with lead scores, badges, response tracking
- **Web should show:** CRM table with lead scoring, pipeline view, response history
- **Key fields:** `leadScores`, `badges`, `responseStatus`, `isBuilder`, `assessmentAnswers`

### Tab: Financials
- **Firestore:** `transactions` (all, not just ADY), `stores`
- **iOS shows:** Transaction list with type filter, store filter, add transaction form
- **Web should show:** Financial dashboard with transaction table, store filter, export to CSV
- **Key fields:** `type`, `amount`, `currency`, `storeId`, `category`, `vendor`, `fundingSource`

### Tab: Stores
- **Firestore:** `stores`
- **iOS shows:** Store list with status, location, owner
- **Web should show:** Store management table with same fields, edit/add forms
- **Key fields:** `name`, `status`, `ownerUserId`, `address`, `city`, `country`, `category`

### Tab: Roles
- **Firestore:** `systemUsers` (filtered by role)
- **iOS shows:** User list grouped by role, role badges, edit role
- **Web should show:** User management table with role assignment, bulk role changes
- **Key fields:** `role` (enum: admin, manager, investor, accountant, psut-advisor, advisor, employee, building-manager, student, client, candidate), `displayName`, `email`

### Tab: My Time (Time Tracking)
- **Firestore:** `timeEntries`
- **iOS shows:** Clock in/out, time log, weekly summary
- **Web should show:** Time tracking dashboard, weekly timesheets, approval workflow
- **Key fields:** `userId`, `clockIn`, `clockOut`, `duration`, `projectId`, `note`

### Tab: Buildings
- **Firestore:** `buildings`, `buildingIssues`
- **iOS shows:** Building list, issue reports, maintenance status
- **Web should show:** Building management dashboard, issue tracker, maintenance schedule
- **Key fields:** `name`, `address`, `issues`, `status`

### Tab: My Track (Student)
- **Firestore:** `studentProgress`, `achievements`, `courses`
- **iOS shows:** Student dashboard with progress, achievements, course status
- **Web should show:** Student portal with same progress tracking, achievement badges
- **Key fields:** `studentId`, `courseId`, `progress`, `completedAt`, `achievementId`

### Tab: Achievements
- **Firestore:** `achievements`, `userAchievements`
- **iOS shows:** Achievement grid with earned/unearned status, progress bars
- **Web should show:** Achievement gallery, leaderboard, badge display
- **Key fields:** `title`, `description`, `icon`, `criteria`, `earnedAt`

### Tab: Reports
- **Firestore:** `transactions`, `periods`, `periodSummaries`, `sharedViews`
- **iOS shows:** Report wizard, PDF export, shareable link generation
- **Web should show:** Report builder, PDF/CSV export, shared view page (public)
- **Key fields:** `sharedViews` collection with `token`, `expiresAt`, `dataSnapshot`

### Tab: Screenshot (Internal Tool)
- **iOS only** — no web equivalent needed
- Used for generating App Store / marketing screenshots with masked sensitive data

### Tab: LaTeX / Carousel
- **iOS only** — internal tools, no web sync needed

---

## 3. Shared Collections Reference

Both iOS and web must read/write these Firestore collections:

| Collection | Purpose | Key consumers |
|---|---|---|
| `systemUsers` | User accounts, roles, auth | Settings, Roles tab |
| `projects` | Project metadata | Dashboard, Projects |
| `transactions` | All financial transactions | Financials, ADY, Payments |
| `periods` | Budget periods | ADY, Budget |
| `periodSummaries` | Computed period totals | ADY overview, Reports |
| `documents` | Project documents/evidence | ADY documents |
| `stores` | Store management | Stores, Financials |
| `crm_contacts` | CRM + submissions | CRM, Submissions |
| `ady_pdf_meta` | Receipt analysis metadata | ADY transactions |
| `sharedViews` | Shareable report links | Reports |
| `timeEntries` | Time tracking | My Time |
| `buildings` | Building management | Buildings |
| `achievements` | Achievement definitions | Achievements |
| `userAchievements` | Earned achievements | My Track, Achievements |
| `studentProgress` | Student course progress | My Track |

---

## 4. Presentation Rules

- **Same data, different layout:** iOS uses tab bar + scroll views. Web uses sidebar navigation + data tables + wider charts.
- **Role-based visibility:** Both platforms must respect `systemUsers.role` to show/hide tabs. Use `isAvailableForRole` logic (iOS) and mirror it in web route guards.
- **Real-time updates:** Both platforms should use Firestore listeners (snapshot listeners on iOS, onSnapshot on web) so changes on one platform appear instantly on the other.
- **Currency:** All amounts stored in original `currency` field. Display in JOD by default. iOS uses `amountJOD` computed property for consolidated totals — web should implement the same conversion logic.
- **Evidence files:** Stored as base64 in `evidenceFiles` array on the transaction document. Both platforms display them the same way (images inline, PDFs with viewer).

---

## 5. Marketing Strategy & User Types

### Target Audience (Two Doors)

**Door 1: Students**
- Fresh graduates and university students looking for career direction
- Two packages offered:
  - **Career Roadmap** — guidance session to map out career path
  - **Interview Success** — mock interview preparation
- $3 booking fee for a 15-minute introductory call
- Goal: Convert to long-term community members

**Door 2: Companies**
- Businesses looking for tech talent, AI development, or project consulting
- Free "Let's Talk" form — collects:
  - Company name
  - Work email (validated as company email, not gmail/personal)
  - Project description
  - Help needed (checkboxes: AI Development, Design, Consulting, Hiring, Other)
  - Preferred schedule
- No upfront cost — relationship-building first, outcome-based packages later

### Marketing Principles
- **No coffee metaphor** — removed entirely from all copy
- **No emojis** in UI text
- **No fluff or buzzwords** — short, direct, factual language
- **No "forever" or "digest" language**
- **Clear calls to action:** "Book a call — $3" for students, "Let's Talk" for companies
- **Minimal packages:** Only show what's relevant to the user's door
- **Community retention:** After initial call, keep users engaged through achievements, progress tracking, and community access

### User Types in the System

| Role | Who they are | What they see |
|---|---|---|
| `admin` | Internal team (Mohammad) | All tabs, user management, settings |
| `manager` | Project managers | Dashboard, projects, financials, CRM |
| `accountant` | Financial staff | Financials, ADY, budget, transactions |
| `investor` | Stakeholders | Dashboard, financials (read-only) |
| `employee` | Staff members | My Time, buildings, limited dashboard |
| `building-manager` | Facility staff | Buildings, building issues |
| `student` | Students in programs | My Track, achievements |
| `client` | Company contacts | Their projects, shared reports |
| `candidate` | Applicants | Submissions status, their applications |
| `advisor` / `psut-advisor` | Academic advisors | Student progress, project overview |

### Web Team Action Items
1. Build the Discover page with two doors (Student / Company) per the spec above
2. Implement company email validation on the "Let's Talk" form
3. Implement $3 booking fee flow for students (Stripe or Tap — pending decision)
4. Mirror the iOS tab structure in the web portal sidebar
5. Use the same Firestore collections listed in section 3
6. Implement real-time listeners so iOS and web stay in sync
7. Add the new `transactions` fields (`label`, `recipientName`, `isRecurring`, `recurrenceFrequency`) to the web transaction forms and displays
8. Build a Payments view on web that groups transactions by label (matching the iOS Payments tab)
