# ADY Project — Complete Guide for Web Developers

> **Purpose:** This document describes every data model, Firestore collection, calculation, and UI view in the ADY (Accounting & Financial Dashboard) project on iOS. The web portal should match these exactly so both platforms stay in sync.

---

## 1. Firestore Collections

All ADY collections use the `ady_` prefix:

| Collection Name | Type | Purpose |
|---|---|---|
| `ady_transactions` | `Transaction` | All financial transactions (expenses + income) |
| `ady_periods` | `Period` | Budgeting windows (months or custom date ranges) |
| `ady_summaries` | `PeriodSummary` | Pre-computed per-period and per-month totals |
| `ady_meetings` | `MeetingNote` | Meeting notes with Sari Awwad, linked to periods |

The iOS app also reads from unprefixed collections (`transactions`, `periods`) as a fallback for legacy data. The web app should write to the `ady_` prefixed collections.

---

## 2. Data Models

### 2.1 Transaction (`ady_transactions`)

```
{
  id: string (Firestore doc ID),
  projectId: string | null,         // should be set to ADY project ID
  storeId: string | null,           // legacy field
  type: string,                     // "payment" | "bill" | "check" | "cost" | "revenue"
  category: string | null,          // e.g. "Marketing", "Equipment"
  amount: number,                   // original amount in the transaction's currency
  currency: string,                 // "JOD" | "USD"
  date: Timestamp | string,         // transaction date
  note: string | null,
  description: string | null,       // human-facing title (shown in UI)
  periodId: string | null,          // explicit link to a Period
  attachmentUrl: string | null,
  evidenceFiles: [EvidenceFile] | null,  // base64-encoded PDFs/images
  installmentId: string | null,
  vendor: string | null,
  subscriptionId: string | null,
  fundingSource: string | null,     // e.g. "psut", "self", "other"
  evidenceAttached: boolean | null,
  notes: string | null,
  createdAt: Timestamp | null,
  updatedAt: Timestamp | null
}
```

**Transaction Type Classification:**

| Type | Is Expense? | Is Income? | Icon |
|---|---|---|---|
| `cost` (or `expense`) | YES | no | tag.fill |
| `bill` | YES | no | doc.text.fill |
| `payment` | no | YES | creditcard.fill |
| `check` | no | YES | banknote.fill |
| `revenue` (or `income`) | no | YES | banknote.fill |

**Key computed properties:**
- `amountJOD` = amount converted to JOD. Conversion rate: **1 USD = 0.707 JOD**
- `isExpense` = type is `bill` or `cost`
- `isIncome` = type is `payment`, `check`, or `revenue`
- `resolvedCategory` = category if non-empty, otherwise the type's display name
- `displayTitle` = description if non-empty, otherwise resolvedCategory

### 2.2 Period (`ady_periods`)

```
{
  id: string (Firestore doc ID),
  storeId: string | null,
  projectId: string | null,
  name: string,                     // e.g. "July 2026"
  startDate: Timestamp | string,
  endDate: Timestamp | string,
  isClosed: boolean | null,         // legacy — use status instead
  status: string,                   // "planned" | "active" | "completed"
  plannedItems: [PlannedItem] | null,  // per-category planned lines
  plannedTotal: number | null,      // simple planned total (when no line items)
  fundName: string | null,          // e.g. "PSUT Venture Lab"
  fundTotal: number | null,         // e.g. 10000
  notes: string | null,
  allocatedFunds: number | null,    // web app's field for planned budget
  spentAmount: number | null,       // denormalised — iOS recomputes on save
  revenue: number | null,           // denormalised
  netPosition: number | null,       // denormalised
  createdAt: Timestamp | null,
  updatedAt: Timestamp | null
}
```

**PlannedItem:**
```
{
  id: string,
  category: string,
  amount: number,
  note: string | null
}
```

**Status resolution:**
- If `status` is set, use it directly
- If `status` is null, fall back to `isClosed`: `isClosed == true` → `completed`, otherwise → `active`
- Decoding is lenient: "closed" → `completed`, "done" → `completed`, unknown → `active`

**Planned total resolution (priority order):**
1. If `plannedItems` is non-empty: sum of all item amounts
2. Else if `plannedTotal` is set: use it
3. Else if `allocatedFunds` is set: use it
4. Else: 0

### 2.3 MeetingNote (`ady_meetings`)

```
{
  id: string (Firestore doc ID),
  projectId: string | null,
  periodId: string | null,          // links to a Period
  meetingDate: Timestamp | null,
  mainPoints: string | null,        // main points discussed
  decisions: string | null,         // decisions / outcomes
  createdBy: string | null,
  createdAt: Timestamp | null
}
```

### 2.4 PeriodSummary (`ady_summaries`)

```
{
  id: string,
  projectId: string | null,
  periodId: string | null,          // non-null for period summaries
  monthKey: string | null,          // "yyyy-MM" for monthly roll-ups
  label: string,                    // display label
  totalExpense: number,
  totalIncome: number,
  netPosition: number,              // totalIncome - totalExpense
  transactionCount: number,
  byType: [SummaryTypeEntry],
  byCategory: [SummaryCategoryEntry],
  computedAt: Timestamp | null
}
```

---

## 3. Calculations

### 3.1 Currency Conversion
- Fixed rate: **1 USD = 0.707 JOD**
- All sums use `amountJOD` (amount converted to JOD) so mixed-currency transactions can be totaled

### 3.2 Transaction → Period Assignment
A transaction belongs to a period if:
1. Its `date` falls within the period's `[startDate, endDate]` range (inclusive of the entire end day), OR
2. Its `periodId` explicitly matches the period's ID

Date range is the **primary** rule. A transaction shows under the period it belongs to by date, even if `periodId` points elsewhere.

### 3.3 Per-Period Calculations

| Calculation | Formula |
|---|---|
| `actualExpense` | Sum of `amountJOD` for all transactions in period where `isExpense == true` |
| `actualIncome` | Sum of `amountJOD` for all transactions in period where `isIncome == true` |
| `netCash` | `actualIncome - actualExpense` (negative = overspend) |
| `plannedTotalResolved` | See priority order above (plannedItems sum → plannedTotal → allocatedFunds → 0) |
| `variance` | `plannedTotalResolved - actualExpense` |
| `avgDailySpend` | `actualExpense / max(1, days in period)` |
| `avgMonthlySpend` | `actualExpense / max(1, months in period)` |

### 3.4 Carry-Over Balance (Across All Periods)

Periods are sorted chronologically by `startDate`. For each period:

```
carryIn  = cumulative balance from all prior periods
periodNet = actualIncome - actualExpense (this period)
carryOut = carryIn + periodNet
```

- `carryOut < 0` → **debt** (shown in red)
- `carryOut >= 0` → **surplus** (shown in green)
- The final period's `carryOut` is the project's overall balance

### 3.5 PSUT Venture Lab Grant Calculations

| Field | Formula |
|---|---|
| `fundTotal` | Stored in `@AppStorage("psut_fund_total")`, default 10000 JOD |
| `fundName` | Stored in `@AppStorage("psut_fund_name")`, default "PSUT Venture Lab" |
| `psutReceived` | Sum of `amountJOD` for income transactions where `fundingSource == "psut"` |
| `psutSpent` | Sum of `amountJOD` for expense transactions where `fundingSource == "psut"` |
| `inCash` | `psutReceived - psutSpent` (can be **negative** — shows in red) |
| `toReceive` | `max(0, fundTotal - psutReceived)` |
| `receivedPct` | `psutReceived / fundTotal * 100` |

**Progress bar segments** (always sum to the grant total, no overflow):
- Spent segment: `min(psutSpent, fundTotal)` — shown in primary color
- Cash segment: `max(0, min(psutReceived, fundTotal) - spentSeg)` — shown in accent color (or red if inCash < 0)
- Pending segment: `fundTotal - spentSeg - cashSeg` — shown in amber

### 3.6 Project-Level Totals

| Field | Formula |
|---|---|
| `totalSpent` | Sum of `amountJOD` for all expense transactions |
| `totalIncome` | Sum of `amountJOD` for all income transactions |
| `isOverspent` | `totalSpent > totalIncome` |
| `overspendAmount` | `max(0, totalSpent - totalIncome)` |

### 3.7 Spending Forecast (for planned periods)

Uses the last 3 completed periods as historical basis:
1. Compute average daily spend from each historical period
2. Average those daily rates
3. Multiply by the target period's duration in days
4. Confidence: `min(1.0, historicalCount / 3.0 * 0.7 + 0.3)`
5. If no completed periods exist, falls back to overall average daily spend with 0.3 confidence

### 3.8 Monthly Buckets

Transactions are grouped by `yyyy-MM` key. Each bucket has:
- `title`: "MMMM yyyy" (e.g. "July 2026")
- `expense`: sum of expense amounts
- `income`: sum of income amounts
- `net`: `income - expense`
- `count`: transaction count
- `byType`: per-type breakdown

### 3.9 Unassigned Transactions

A transaction is "unassigned" if it doesn't fall within any period's date range AND has no matching `periodId`. These need manual assignment.

---

## 4. iOS App Views (Tabs)

The ADY project has 6 tabs in the iOS app:

### 4.1 Dashboard (Overview)
- **PSUT Grant Card**: Three-segment progress bar (Spent / In cash / To receive)
  - Shows `fundName`, `fundTotal`, `psutReceived`, `psutSpent`, `inCash`, `toReceive`
  - `inCash` can go negative (red) to show overspend
- **Quick Stats**: Transaction count, Period count, Total Spent, Total Income
- **Overspend Alert**: Shows when `totalSpent > totalIncome`
- **Unassigned Alert**: Shows count of transactions not assigned to any period
- **Recent Transactions**: Latest 5 transactions
- **Spending Insights**: Breakdown by funding source + top categories
- **Periods Summary**: Mini overview of all periods
- **Monthly Summary**: Last 4 months with expense/income

### 4.2 Budget (Periods)
- **Periods Summary Table**: Table with columns: Period, Planned, Received, Spent, Net, Carry-over
  - Color-coded: green for surplus, red for deficit/debt
  - Totals row at bottom
- **Period Cards**: One per period showing name, date range, status, planned vs actual, transaction count
- **Carry-over Timeline**: Chronological running balance across all periods
- **Unassigned Transactions**: List of transactions not in any period

### 4.3 Transactions
- Full transaction explorer with search, filter, sort, group, and bulk move

### 4.4 Reports
- **PSUT Venture Lab Report**: Generates a PDF matching the Venture Lab template:
  - 01 Project Information (auto-filled)
  - 02 Meeting with Sari Awwad (auto-filled from `ady_meetings`)
  - 03 Achievements & Work Completed
  - 04 Challenges & Risks
  - 05 Support & Assistance Needed (checkboxes)
  - 06 Budget & Spending (auto-filled: Total In/Out/Balance, spending log, planned spending)
  - 07 Budget Overview & Summary
  - Opens in QuickLook preview (not direct share)
- **Per-period Venture Lab Reports**: One card per period with PDF export
- **Period Reports**: Per-period PDF and CSV export
- **Monthly Reports**: Per-month PDF and CSV export

### 4.5 Achievements
- Timeline of project milestones and achievements

### 4.6 Documents
- Project document library

---

## 5. Period Editor (Create/Edit Period)

Fields available:
- **Name** (required)
- **Start Date** (required)
- **End Date** (required, must be >= start)
- **Status**: planned / active / completed (segmented picker)
- **Fund name** (optional, default "PSUT Venture Lab")
- **Notes** (optional)
- **Planned Total** (numeric field — the planned budget for this period)
- **Planned Spend Lines** (optional per-category breakdown: category + amount)

On save:
1. If planned line items exist and have categories: `plannedTotal` = sum of line items
2. Else if `plannedTotalAmount` > 0: `plannedTotal` = that value
3. Else: `plannedTotal` = nil
4. `allocatedFunds` is set to `plannedTotalResolved` if not already set
5. `spentAmount`, `revenue`, `netPosition` are recomputed and saved

---

## 6. Meeting with Sari

- Stored in `ady_meetings` collection
- Linked to a period via `periodId`
- Fields: `meetingDate`, `mainPoints`, `decisions`
- Auto-included in the Venture Lab PDF report when they exist for the period
- Can be added/edited from both the Budget toolbar and the Reports tab

---

## 7. What the Web Portal Needs to Fix

1. **Collection names**: Use `ady_` prefix for all ADY collections
2. **Transaction types**: Ensure `type` field uses `payment`, `bill`, `check`, `cost`, `revenue`
3. **Expense classification**: `bill` and `cost` = expense; `payment`, `check`, `revenue` = income
4. **Currency**: All amounts should specify `currency` ("JOD" or "USD"). The app converts to JOD at 0.707 rate
5. **Period status**: Use `planned`, `active`, `completed` (not "closed")
6. **Planned total**: Support both `plannedTotal` (simple number) and `plannedItems` (per-category lines). If `plannedItems` exist, their sum takes priority
7. **Allocated funds**: `allocatedFunds` is the web's equivalent of `plannedTotal` — iOS reads it as a fallback
8. **Funding source**: Set `fundingSource: "psut"` for PSUT Venture Lab transactions so they're counted in the grant card
9. **Meeting notes**: Write to `ady_meetings` with `periodId`, `meetingDate`, `mainPoints`, `decisions`
10. **Negative cash**: The dashboard shows negative "In cash" in red when `psutSpent > psutReceived` — the web should do the same
11. **Carry-over**: Display running balance across all periods chronologically — negative = debt (red), positive = surplus (green)
12. **Period summary table**: Show a table with per-period: Planned, Received (income), Spent (expense), Net, Carry-over, with a totals row
13. **Venture Lab Report**: The web should offer the same PSUT Venture Lab PDF template with all 7 sections

---

## 8. Firestore Document ID Convention

- All documents use Firestore auto-generated IDs
- The `@DocumentID` annotation in Swift reads the Firestore doc ID automatically
- The web app should not override IDs — let Firestore generate them

---

## 9. Date Handling

The iOS app is lenient with dates — it accepts:
- Firestore Timestamp objects
- ISO 8601 strings
- Epoch milliseconds (numbers)
- Maps with `_seconds` and `_nanoseconds` (Firestore Timestamp serialized)

The web app should use Firestore Timestamps for all date fields for best compatibility.
