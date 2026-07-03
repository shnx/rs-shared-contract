# iOS → Web: Full Transaction Calculation Code + Carry-Over Fix (July 3)

**From:** iOS team
**Re:** Here's our actual Swift code for every calculation. Learn from it. Also: you're missing carry-over between periods.

---

## THE BUG: Missing Carry-Over Between Periods

Your web portal is likely calculating each period in isolation. That's wrong. **Periods are cumulative** — the net from period 1 carries into period 2, period 2's net adds to that, and so on. If period 1 overspends by JD 500, period 2 starts with a JD 500 debt.

Here's our carry-over code:

```swift
// BudgetPeriodsView.swift — lines 549-559

func carryOverTimeline() -> [PeriodCarryOver] {
    let sorted = periods.sorted { $0.startDate < $1.startDate }
    var cumulative: Decimal = 0
    return sorted.map { p in
        let net = netCash(in: p)           // income - expense for THIS period
        let carryIn = cumulative            // what we brought from prior periods
        cumulative += net                   // add this period's net
        return PeriodCarryOver(
            period: p,
            periodNet: net,
            carryIn: carryIn,
            carryOut: cumulative            // new running balance
        )
    }
}

// isDebt = true when carryOut < 0
```

**In JavaScript:**
```javascript
function carryOverTimeline(periods, transactions) {
  const sorted = [...periods].sort((a, b) => a.startDate - b.startDate);
  let cumulative = 0;
  return sorted.map(p => {
    const periodTx = transactions.filter(t => isInPeriod(t, p));
    const income = periodTx.filter(t => isIncome(t)).reduce((s, t) => s + amountJOD(t), 0);
    const expense = periodTx.filter(t => isExpense(t)).reduce((s, t) => s + amountJOD(t), 0);
    const net = income - expense;
    const carryIn = cumulative;
    cumulative += net;
    return {
      periodId: p.id,
      periodName: p.name,
      periodNet: net,
      carryIn: carryIn,
      carryOut: cumulative,
      isDebt: cumulative < 0
    };
  });
}
```

---

## How We Classify Transactions

```swift
// BudgetPeriodsView.swift — lines 30-38

extension Transaction {
    var amountJOD: Decimal { currency.convert(amount: amount, to: .JOD) }

    // EXPENSE: money leaving the fund
    var isExpense: Bool { type == .bill || type == .cost }

    // INCOME: money coming in
    var isIncome: Bool { type == .payment || type == .check || type == .revenue }
}
```

**Critical:** `payment` is INCOME, not expense. If you treat `payment` as expense, all your numbers are wrong.

```javascript
// JavaScript equivalent
function isExpense(t) {
  return t.type === 'bill' || t.type === 'cost';
}

function isIncome(t) {
  return t.type === 'payment' || t.type === 'check' || t.type === 'revenue';
}

function amountJOD(t) {
  if (t.currency === 'USD') return t.amount * 0.707;
  return t.amount;
}
```

---

## How We Assign Transactions to Periods

**Date range is PRIMARY. `periodId` is SECONDARY.** A transaction belongs to a period if its date falls within the period's range, OR it was explicitly assigned. Date range wins.

```swift
// BudgetPeriodsView.swift — lines 404-417

func dateInRange(_ t: Transaction, _ period: Period) -> Bool {
    let cal = Calendar.current
    let start = cal.startOfDay(for: period.startDate)
    // endExclusive = start of the day AFTER period.endDate
    // (so the entire end day is included)
    let endExclusive = cal.date(byAdding: .day, value: 1,
                                to: cal.startOfDay(for: period.endDate)) ?? period.endDate
    return t.date >= start && t.date < endExclusive
}

func transactions(in period: Period) -> [Transaction] {
    transactions.filter {
        dateInRange($0, period) || ($0.periodId != nil && $0.periodId == period.id)
    }
}
```

```javascript
// JavaScript equivalent
function isInPeriod(t, period) {
  const start = new Date(period.startDate);
  start.setHours(0, 0, 0, 0);

  const endExclusive = new Date(period.endDate);
  endExclusive.setHours(0, 0, 0, 0);
  endExclusive.setDate(endExclusive.getDate() + 1);

  const tDate = new Date(t.date);
  return (tDate >= start && tDate < endExclusive) ||
         (t.periodId && t.periodId === period.id);
}
```

**Common mistake:** If you only use `periodId` and ignore date range, transactions without `periodId` won't appear in any period. You'll have "missing" transactions.

---

## How We Calculate Period Totals

```swift
// BudgetPeriodsView.swift — lines 419-433

func actualExpense(in period: Period) -> Decimal {
    transactions(in: period)
        .filter { $0.isExpense }
        .reduce(Decimal(0)) { $0 + $1.amountJOD }
}

func actualIncome(in period: Period) -> Decimal {
    transactions(in: period)
        .filter { $0.isIncome }
        .reduce(Decimal(0)) { $0 + $1.amountJOD }
}

func netCash(in period: Period) -> Decimal {
    actualIncome(in: period) - actualExpense(in: period)
}
```

---

## How We Calculate Project-Level Totals

```swift
// BudgetPeriodsView.swift — lines 362-383

// PSUT-specific: only transactions with fundingSource == "psut"
var psutSpent: Decimal {
    transactions
        .filter { $0.isExpense && ($0.fundingSource ?? "").lowercased() == "psut" }
        .reduce(Decimal(0)) { $0 + $1.amountJOD }
}

var psutReceived: Decimal {
    transactions
        .filter { $0.isIncome && ($0.fundingSource ?? "").lowercased() == "psut" }
        .reduce(Decimal(0)) { $0 + $1.amountJOD }
}

// Project-wide: ALL transactions regardless of funding source
var totalSpent: Decimal {
    transactions.filter { $0.isExpense }.reduce(Decimal(0)) { $0 + $1.amountJOD }
}

var totalIncome: Decimal {
    transactions.filter { $0.isIncome }.reduce(Decimal(0)) { $0 + $1.amountJOD }
}
```

**`totalSpent` ≠ `psutSpent`.** `totalSpent` includes ALL expenses. `psutSpent` only includes expenses where `fundingSource == "psut"`.

---

## How In Cash Can Be NEGATIVE

```swift
// ADYProjectView.swift — lines 471-477

private var fundCard: some View {
    let total = Decimal(fundTotal)        // e.g. 10,000
    let received = viewModel.psutReceived  // e.g. 3,000
    let spent = viewModel.psutSpent        // e.g. 3,500

    // Can go negative to show overspend clearly.
    // If received=3000 and spent=3500, inCash = -500
    let inCash = received - spent          // -500

    // How much of the grant we still expect to receive.
    let toReceive = max(Decimal(0), total - received)  // 7,000
}

// Display: show inCash in RED when negative
legendStat("In cash", Money.string(inCash),
           inCash >= 0 ? cashColor : AppThemeColors.error)
```

**Your bug:** You have `In Cash = max(0, psutReceived - psutSpent)`. Remove the `max(0, ...)`. Let it go negative. Show it in red.

```javascript
// CORRECT JavaScript
const inCash = psutReceived - psutSpent;  // can be negative
const inCashColor = inCash >= 0 ? 'green' : 'red';
```

---

## How We Calculate the Progress Bar (When Overspent)

```swift
// ADYProjectView.swift — lines 481-489

// Bar segments that ALWAYS sum to the grant (no overflow on overspend)
let spentSeg = min(d(spent), totalD)           // cap at total
let cashSeg = max(0, min(d(received), totalD) - spentSeg)
let pendingSeg = max(0, totalD - spentSeg - cashSeg)
```

When overspent:
- `spentSeg` = full grant (capped)
- `cashSeg` = 0 (no cash left)
- `pendingSeg` = 0 (or shows the remaining grant to receive)
- The "In cash" number shows negative in red below the bar

---

## How We Calculate Monthly Buckets

```swift
// BudgetPeriodsView.swift — lines 472-491

var monthlyBuckets: [MonthBucket] {
    var buckets: [String: MonthBucket] = [:]
    let calendar = Calendar.current
    for t in transactions {
        let key = DateFmt.monthKey(t.date)           // "2026-06"
        let startOfMonth = calendar.dateInterval(of: .month, for: t.date)?.start ?? t.date
        var bucket = buckets[key] ?? MonthBucket(
            id: key, title: DateFmt.monthTitle(t.date), sortDate: startOfMonth
        )
        if t.isExpense { bucket.expense += t.amountJOD }
        if t.isIncome  { bucket.income  += t.amountJOD }
        bucket.count += 1
        buckets[key] = bucket
    }
    return Array(buckets.values).sorted { $0.sortDate > $1.sortDate }
}
```

---

## How We Calculate Cash Flow Timeline (Running Balance Within a Period)

```swift
// BudgetPeriodsView.swift — lines 506-514

func cashFlowTimeline(in period: Period) -> [CashFlowEvent] {
    let items = transactions(in: period).sorted { $0.date < $1.date }
    var balance: Decimal = 0
    return items.map { t in
        let delta = t.isIncome ? t.amountJOD : -t.amountJOD
        balance += delta
        return CashFlowEvent(
            transaction: t,
            delta: delta,
            runningBalance: balance,
            isOverspend: balance < 0
        )
    }
}
```

---

## How We Calculate Unassigned Transactions

Transactions that don't fall in ANY period — these are the ones you're probably missing:

```swift
// BudgetPeriodsView.swift — lines 435-439

var unassignedTransactions: [Transaction] {
    transactions.filter { t in
        !periods.contains { dateInRange(t, $0) || (t.periodId != nil && t.periodId == $0.id) }
    }
}
```

**Check this on your web portal.** If you have unassigned transactions, they won't appear in any period view but they DO count in project-level totals (`totalSpent`, `totalIncome`). This could explain why your period totals don't add up to the project total.

---

## How We Calculate Planned vs Actual

```swift
// BudgetPeriodsView.swift — lines 455-468

func categoryBreakdown(for period: Period) -> [CategoryAggregate] {
    var planned: [String: Decimal] = [:]
    for item in period.plannedItems ?? [] {
        planned[item.category, default: 0] += item.amount
    }
    var actual: [String: Decimal] = [:]
    for t in transactions(in: period) where t.isExpense {
        actual[t.resolvedCategory, default: 0] += t.amountJOD
    }
    let categories = Set(planned.keys).union(actual.keys).sorted()
    return categories.map {
        CategoryAggregate(
            category: $0,
            planned: planned[$0] ?? 0,
            actual: actual[$0] ?? 0
        )
    }
}

// Variance = planned - actual (positive = under budget, negative = over budget)
```

---

## How We Calculate the Period's Planned Total

```swift
// Models.swift — lines 602-608

var plannedTotalResolved: Decimal {
    if let items = plannedItems, !items.isEmpty {
        return items.reduce(Decimal(0)) { $0 + $1.amount }
    }
    return plannedTotal ?? allocatedFunds ?? 0
}
```

Priority: `plannedItems` sum → `plannedTotal` → `allocatedFunds` → 0

---

## How We Calculate Overspend

```swift
// BudgetPeriodsView.swift — lines 576-579

var isOverspent: Bool { totalSpent > totalIncome }

var overspendAmount: Decimal {
    max(0, totalSpent - totalIncome)
}
```

**Note:** This is project-level overspend (ALL expenses vs ALL income). This is different from PSUT overspend (`psutSpent > psutReceived`).

---

## How We Calculate Average Daily/Monthly Spend

```swift
// BudgetPeriodsView.swift — lines 517-528

func avgDailySpend(in period: Period) -> Decimal {
    let cal = Calendar.current
    let days = max(1, (cal.dateComponents([.day], from: period.startDate, to: period.endDate).day ?? 0) + 1)
    return actualExpense(in: period) / Decimal(days)
}

func avgMonthlySpend(in period: Period) -> Decimal {
    let cal = Calendar.current
    let months = max(1, (cal.dateComponents([.month], from: period.startDate, to: period.endDate).month ?? 0) + 1)
    return actualExpense(in: period) / Decimal(months)
}
```

---

## The Period Model (Full Schema)

```swift
// Models.swift — lines 512-608

struct Period: Codable, Identifiable {
    @DocumentID var id: String?
    var storeId: String?
    var projectId: String?
    var name: String
    var startDate: Date
    var endDate: Date
    var isClosed: Bool?           // legacy
    var status: PeriodStatus?     // "planned" | "active" | "completed"
    var plannedItems: [PlannedItem]?
    var plannedTotal: Decimal?
    var fundName: String?
    var fundTotal: Decimal?
    var notes: String?
    var createdAt: Date?
    var updatedAt: Date?

    // Web-app fields
    var allocatedFunds: Decimal?
    var spentAmount: Decimal?     // web-maintained; iOS recomputes
    var revenue: Decimal?         // web-maintained; iOS recomputes
    var netPosition: Decimal?     // web-maintained; iOS recomputes

    var effectiveStatus: PeriodStatus {
        if let status { return status }
        return (isClosed == true) ? .completed : .active
    }

    var plannedTotalResolved: Decimal {
        if let items = plannedItems, !items.isEmpty {
            return items.reduce(Decimal(0)) { $0 + $1.amount }
        }
        return plannedTotal ?? allocatedFunds ?? 0
    }
}
```

**Important:** Don't trust `spentAmount`, `revenue`, `netPosition` from the web. Recompute them from transactions. These are cached values that can be stale.

---

## The Transaction Model (Full Schema)

```swift
// Models.swift — lines 216-357

struct Transaction: Codable, Identifiable {
    @DocumentID var id: String?
    var projectId: String?
    var storeId: String?
    var type: TransactionType       // payment, bill, check, cost, revenue
    var category: String?
    var amount: Decimal
    var currency: Currency          // JOD or USD
    var date: Date
    var note: String?
    var attachmentUrl: String?
    var createdAt: Date?
    var updatedAt: Date?
    var description: String?
    var periodId: String?
    var evidenceFiles: [EvidenceFile]?
    var installmentId: String?
    var vendor: String?
    var subscriptionId: String?
    var fundingSource: String?      // "psut", "own-capital", "revenue"
    var evidenceAttached: Bool?
    var notes: String?
}

enum TransactionType: String, Codable {
    case payment    // INCOME
    case bill       // EXPENSE
    case check      // INCOME
    case cost       // EXPENSE
    case revenue    // INCOME
}

enum Currency: String, Codable {
    case JOD
    case USD
    // 1 USD = 0.707 JOD
    func convert(amount: Decimal, to target: Currency) -> Decimal {
        if self == target { return amount }
        switch (self, target) {
        case (.USD, .JOD): return amount * 0.707
        case (.JOD, .USD): return amount / 0.707
        default: return amount
        }
    }
}
```

---

## Summary: What You Need to Fix

| Bug | What you're doing | What you should do |
|---|---|---|
| **Missing carry-over** | Each period calculated in isolation | Cumulative net flows forward |
| **In Cash clamped** | `max(0, received - spent)` | `received - spent` (let it go negative, show red) |
| **Payment as expense** | Treating `payment` as expense | `payment` is INCOME |
| **Only using periodId** | Filtering by `periodId` only | Use date range first, `periodId` as fallback |
| **Trusting cached fields** | Using `spentAmount`, `revenue` from Firestore | Recompute from transactions |
| **Missing unassigned tx** | Not showing transactions outside any period | Check `unassignedTransactions` and display them |

---

## Our Full Code Files for Reference

| File | What's in it | Lines |
|---|---|---|
| `BudgetPeriodsView.swift` | All calculations: transactions, periods, carry-over, monthly buckets, cash flow, forecasts, CSV export | 3,465 |
| `ADYProjectView.swift` | ADY project UI: dashboard, fund card (with negative inCash), reports tab, shareable link, Venture Lab report export | 2,115 |
| `Models.swift` | All data models: Transaction, Period, PlannedItem, EvidenceFile, PortalDocument, MeetingNote | 1,311 |
| `ReportPDFView.swift` | Venture Lab PDF report builder: all sections, spending table, planned spending | 483 |
| `AchievementsView.swift` | Achievements timeline with transaction linkage, evidence access | 681 |
| `FirestoreRepository.swift` | All Firestore repositories, collection prefixing, merge logic | 667 |

All code is on `devin-1` branch. Clone and read it if you need the full picture.
