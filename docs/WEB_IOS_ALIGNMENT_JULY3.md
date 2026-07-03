# Web → iOS: Portal Aligned to Match iOS Exactly (July 3)

**Date:** July 3, 2026
**From:** Web team
**Re:** We've copied your tab structure, calculations, and design — please review and share anything we missed

---

## What We Did

### 1. Tab Structure — Now Matches iOS Exactly

We trimmed the web portal to match your tab layout 1:1. Extra tabs moved to an internal "Tools & Admin" project (web-only, not visible to users).

| Project | Tabs (Web = iOS) |
|---|---|
| **ADY** | Dashboard, Budget, Transactions, Reports, Achievements, Documents |
| **Tools & Website** | Submissions, LaTeX, CRM, Careers, Messages |
| **Riad Shannak** | Buildings, Documents, Achievements |

**Removed from ADY** (moved to internal): Finance, Charts, Investor Portal, Launchpad, Overview, Admin Settings
**Removed from Tools & Website** (moved to internal): Website, Video Tutorials, Video Assignments, Documents

### 2. Financial Calculations — Copied from Your Data Contract

We created a shared `financials.ts` module that implements your exact formulas from `DATA_CONTRACT.md`:

**Transaction classification:**
- Expense = `type == "cost"` OR `type == "bill"`
- Income = `type == "revenue"` OR `type == "payment"` OR `type == "check"`

**Currency:**
- JOD = use as-is
- USD → JOD = multiply by `0.707`

**Project-level metrics:**
| Metric | Formula |
|---|---|
| Total Spent | Σ `amountJOD` where `isExpense` |
| Total Income | Σ `amountJOD` where `isIncome` |
| PSUT Spent | Σ `amountJOD` where `isExpense` AND `fundingSource=="psut"` |
| PSUT Received | Σ `amountJOD` where `isIncome` AND `fundingSource=="psut"` |
| Fund Total | 10,000 JOD (hardcoded, matches your @AppStorage) |
| In Cash | `max(0, psutReceived - psutSpent)` |
| To Receive | `max(0, fundTotal - psutReceived)` |
| Is Overspent | `totalSpent > totalIncome` |
| Overspend Amount | `max(0, totalSpent - totalIncome)` |

**Period-level metrics:**
| Metric | Formula |
|---|---|
| Actual Expense | Σ `amountJOD` where `isExpense` AND in period |
| Actual Income | Σ `amountJOD` where `isIncome` AND in period |
| Net Cash | `actualIncome - actualExpense` |
| Avg Daily Spend | `actualExpense / max(1, daysInPeriod)` |

**Transaction belongs to period if:**
- `transaction.date` is within `[period.startDate, period.endDate]` (inclusive of entire end day), OR
- `transaction.periodId == period.id`

**Carry-over debt:**
- Sort periods by `startDate` ascending
- `periodNet` = `actualIncome - actualExpense`
- `carryIn` = cumulative net from all prior periods
- `carryOut` = `carryIn + periodNet`
- `isDebt` = `carryOut < 0`

### 3. Budget Page — Now Shows iOS Metrics

The Budget page now displays:
- **Row 1:** Fund Total (10,000 JOD), PSUT Received, PSUT Spent, In Cash
- **Row 2:** Total Spent, Total Income, To Receive, Overspend
- **Overspend warning** (red banner when `totalSpent > totalIncome`)
- **Carry-over Debt table** (per period: Net, Carry In, Carry Out, DEBT badge)
- **Period cards** with allocated, spent, revenue, net, % used
- **Funding installments** section

### 4. All Pages Updated

Every page that classified transactions now uses the shared `isExpense`/`isIncome` functions:
- Budget, Transactions, Reports, Charts, InvestorPortal, Overview, Launchpad, Finance

### 5. Firestore Projects Updated

Updated `attachedTabs` for all 4 projects to match the iOS tab lists exactly.

---

## What We Need From You

Since your portal works as desired for ADY, please share:

### 1. Budget Page Layout
- What exactly does the iOS Budget tab show? (summary cards, period list, installment list?)
- Do you show the same Fund Total / PSUT Received / PSUT Spent / In Cash metrics?
- How do you display carry-over debt? (table, cards, inline?)

### 2. Dashboard Tab
- What stats/cards does the iOS Dashboard show for ADY?
- Is it the same metrics as Budget, or different?
- Do you show recent transactions, period summaries, or both?

### 3. Transactions Tab
- Do you show a summary header (total revenue, total costs, net)?
- What filters do you have? (by period, by type, by category?)
- Do you group transactions by period or show a flat list?

### 4. Reports Tab
- What report types do you generate?
- Do you show charts inline or just PDF/CSV export?

### 5. Achievements Tab
- Do you show a vertical timeline with images?
- What fields from `timeline_entries` do you display?

### 6. Period Detail View
- When you tap a period, what do you see?
- Do you show planned vs actual spending breakdown?
- Do you show a cash flow timeline (running balance)?

### 7. Any iOS-Specific Behavior We Should Match
- Swipe-to-delete on transactions?
- Pull-to-refresh?
- Any status badges or colors we should replicate?

---

## Current Web Tab Summary

| Project | Type | Tabs |
|---|---|---|
| ADY | `accounting` | Dashboard, Budget, Transactions, Reports, Achievements, Documents |
| Tools & Website | `website` | Submissions, LaTeX, CRM, Careers, Messages |
| Riad Shannak | `real_estate` | Buildings, Documents, Achievements |
| Tools & Admin | `internal` | (web-only) Dashboard, Overview, Launchpad, Finance, Charts, Investor Portal, User Mgmt, Role Mgmt, Project Mgmt, Testing Lab, Agent Log, Operations, Settings, Student Portal, Student Overview, Student Dashboard, Course Mgmt, Membership Mgmt, Invitation Mgr, CV Builder, LaTeX, Documents, Website, Video Tutorials, Video Assignments |

The "Tools & Admin" project is web-only and not visible in the iOS app. It contains all the extra admin/management tools that don't have iOS equivalents.

---

Let us know what we're missing and we'll align immediately.
