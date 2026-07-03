# ADY Budget Calculations — Exact Spec for Web Developers

> **READ THIS CAREFULLY.** If your numbers don't match iOS, you have a bug in one of these calculations. This document gives you the EXACT formula for every single number shown in the ADY dashboard.

---

## 1. TRANSACTION CLASSIFICATION (Get This Right First)

Every transaction has a `type` field. Here is the EXACT classification:

```
type = "cost"     → isExpense = true,  isIncome = false
type = "bill"     → isExpense = true,  isIncome = false
type = "payment"  → isExpense = false, isIncome = true
type = "check"    → isExpense = false, isIncome = true
type = "revenue"  → isExpense = false, isIncome = true
```

**If you classify `payment` as expense, ALL your numbers will be wrong.**
`payment` means money RECEIVED (income). `bill` and `cost` mean money SPENT (expense).

### Currency Conversion

```
amountJOD = amount  (if currency == "JOD")
amountJOD = amount * 0.707  (if currency == "USD")
amountJOD = amount / 0.707  (if converting from JOD to USD, but we always sum in JOD)
```

**ALL sums use `amountJOD`.** Never sum raw amounts across different currencies.

### Resolved Category

```
resolvedCategory = category  (if category is non-empty)
resolvedCategory = type.displayName  (if category is empty/missing)
```

### Display Title

```
displayTitle = description  (if description is non-empty)
displayTitle = resolvedCategory  (otherwise)
```

---

## 2. PERIOD CALCULATIONS (Each One, Exactly)

For a given period P with startDate and endDate:

### 2.1 Which Transactions Belong to a Period

```
function transactionsInPeriod(P):
  return all transactions where:
    (transaction.date >= P.startDate AND transaction.date < dayAfter(P.endDate))
    OR (transaction.periodId == P.id)
```

The end date is INCLUSIVE of the whole day. So if endDate = July 31, transactions on July 31 at 23:59 count. Implementation: add 1 day to endDate, then check `date < endDate + 1 day`.

**Date range is the PRIMARY rule.** Even if `periodId` points to a different period, if the date falls in period P, the transaction belongs to P.

### 2.2 Actual Expense (for a period)

```
actualExpense(P) = SUM of amountJOD for all transactions in P where isExpense == true
```

### 2.3 Actual Income (for a period)

```
actualIncome(P) = SUM of amountJOD for all transactions in P where isIncome == true
```

### 2.4 Net Cash (for a period)

```
netCash(P) = actualIncome(P) - actualExpense(P)
```

Negative = overspent. Positive = surplus. **This can be negative. Show it in red.**

### 2.5 Planned Total (for a period)

Resolve in this priority order:

```
plannedTotalResolved(P):
  1. If P.plannedItems is non-empty: return SUM of all item.amount
  2. Else if P.plannedTotal is not null: return P.plannedTotal
  3. Else if P.allocatedFunds is not null: return P.allocatedFunds
  4. Else: return 0
```

### 2.6 Variance (for a period)

```
variance(P) = plannedTotalResolved(P) - actualExpense(P)
```

### 2.7 Average Daily Spend

```
daysInPeriod = max(1, days between P.startDate and P.endDate inclusive)
avgDailySpend(P) = actualExpense(P) / daysInPeriod
```

### 2.8 Average Monthly Spend

```
monthsInPeriod = max(1, months between P.startDate and P.endDate inclusive)
avgMonthlySpend(P) = actualExpense(P) / monthsInPeriod
```

---

## 3. CARRY-OVER BALANCE (Across All Periods)

This is the MOST IMPORTANT calculation for showing correct debt/surplus.

```
function carryOverTimeline():
  Sort all periods by startDate ascending (oldest first)
  
  cumulative = 0
  
  for each period P in sorted order:
    periodNet = actualIncome(P) - actualExpense(P)
    carryIn = cumulative
    carryOut = carryIn + periodNet
    cumulative = carryOut
    
    // carryOut < 0 means DEBT (show in red)
    // carryOut >= 0 means SURPLUS (show in green)
```

**The last period's `carryOut` is the project's overall balance.**

### Example:

| Period | Income | Expense | Net | Carry-In | Carry-Out |
|--------|--------|---------|-----|----------|-----------|
| Jan | 2000 | 1500 | +500 | 0 | +500 |
| Feb | 1000 | 1800 | -800 | +500 | -300 |
| Mar | 1500 | 1000 | +500 | -300 | +200 |

- Feb shows -300 in red (debt carried into March)
- Final balance = +200

---

## 4. PSUT VENTURE LAB GRANT CALCULATIONS

The grant card on the dashboard shows these numbers:

```
fundTotal = 10000  (stored in AppStorage, default 10000 JOD)
fundName = "PSUT Venture Lab"

psutReceived = SUM of amountJOD for all transactions where:
  isIncome == true AND fundingSource.lowercase() == "psut"

psutSpent = SUM of amountJOD for all transactions where:
  isExpense == true AND fundingSource.lowercase() == "psut"

inCash = psutReceived - psutSpent
  → CAN BE NEGATIVE. Show in red when negative.
  → DO NOT clamp to zero. We intentionally show negative.

toReceive = max(0, fundTotal - psutReceived)

receivedPct = round(psutReceived / fundTotal * 100)
```

### Progress Bar Segments

The bar always sums to exactly `fundTotal` (no overflow):

```
spentSeg = min(psutSpent, fundTotal)
cashSeg = max(0, min(psutReceived, fundTotal) - spentSeg)
pendingSeg = fundTotal - spentSeg - cashSeg

spentFrac = spentSeg / fundTotal
cashFrac = cashSeg / fundTotal
pendingFrac = pendingSeg / fundTotal
```

Colors:
- Spent segment: primary blue
- Cash segment: green (or RED if inCash < 0)
- Pending segment: amber/yellow

---

## 5. PROJECT-LEVEL TOTALS

```
totalSpent = SUM of amountJOD for ALL transactions where isExpense == true
totalIncome = SUM of amountJOD for ALL transactions where isIncome == true

isOverspent = totalSpent > totalIncome
overspendAmount = max(0, totalSpent - totalIncome)
```

**IMPORTANT:** `totalSpent` includes ALL expense transactions regardless of funding source. `psutSpent` only includes expenses where `fundingSource == "psut"`. These are DIFFERENT numbers.

---

## 6. MONTHLY BUCKETS

```
function monthlyBuckets():
  Group all transactions by monthKey = "yyyy-MM" (derived from transaction.date)
  
  For each month:
    title = "MMMM yyyy" (e.g. "July 2026")
    income = SUM of amountJOD where isIncome
    expense = SUM of amountJOD where isExpense
    net = income - expense
    count = number of transactions
    
  Sort by sortDate descending (most recent first)
```

---

## 7. UNASSIGNED TRANSACTIONS

```
function unassignedTransactions():
  return transactions where:
    NOT (any period P exists where:
      dateInRange(transaction, P) OR transaction.periodId == P.id
    )
```

These are transactions that don't belong to any period by date or explicit assignment.

---

## 8. PERIOD SUMMARY TABLE (Shown in the Budget Tab)

This table appears at the top of the Budget/Periods view:

| Period | Planned | Received | Spent | Net | Carry |
|--------|---------|----------|-------|-----|-------|
| Jan 2026 | 2000 | 2000 | 1500 | +500 | +500 |
| Feb 2026 | 2000 | 1000 | 1800 | -800 | -300 |
| Mar 2026 | 2000 | 1500 | 1000 | +500 | +200 |
| **Total** | **6000** | **4500** | **4300** | **+200** | **+200** |

Column definitions:
- **Planned** = `plannedTotalResolved(P)`
- **Received** = `actualIncome(P)` (all income in the period)
- **Spent** = `actualExpense(P)` (all expenses in the period)
- **Net** = `actualIncome(P) - actualExpense(P)` (green if >= 0, red if < 0)
- **Carry** = `carryOut` from the carry-over timeline (green if >= 0, red if < 0)

Totals row:
- **Total Planned** = sum of all periods' plannedTotalResolved
- **Total Received** = sum of all periods' actualIncome
- **Total Spent** = sum of all periods' actualExpense
- **Total Net** = totalReceived - totalSpent
- **Total Carry** = last period's carryOut

---

## 9. SPENDING FORECAST (For Planned Periods)

```
function forecast(targetPeriod):
  targetDays = days between targetPeriod.startDate and targetPeriod.endDate (inclusive)
  
  historical = last 3 COMPLETED periods (status == "completed") 
               where endDate < targetPeriod.startDate
               sorted by startDate descending
  
  if historical is empty:
    avgDaily = totalSpent / max(1, totalTransactionCount)
    return projectedSpend = avgDaily * targetDays, confidence = 0.3
  
  dailyRates = for each historical period:
    days = days in that period (inclusive)
    spend = actualExpense(period) / days
    income = actualIncome(period) / days
  
  avgDailySpend = average of all dailyRates.spend
  avgDailyIncome = average of all dailyRates.income
  confidence = min(1.0, historicalCount / 3.0 * 0.7 + 0.3)
  
  projectedSpend = avgDailySpend * targetDays
  projectedIncome = avgDailyIncome * targetDays
```

---

## 10. ACHIEVEMENTS / TIMELINE

### Collection: `timeline_entries` (also checks `timeline` and `ady_timeline` as fallback)

### Document Schema:

```json
{
  "title": "string (required)",
  "summary": "string (optional, short one-liner)",
  "description": "string (optional, longer text)",
  "date": "Timestamp (the achievement date)",
  "imageUrl": "string (optional, URL to image in Firebase Storage)",
  "category": "string (optional, e.g. 'Research', 'Development')",
  "projectId": "string (should be set to ADY project ID)",
  "entry_type": "string (see types below)",
  "is_visible": "boolean (default true, set false to hide)",
  "amount": "number (optional, for income/expense entries)",
  "currency": "string ('JOD' or 'USD')",
  "payment_method": "string (optional)",
  "transaction_id": "string (optional, links to a transaction)",
  "created_by": "string (optional, user name)",
  "createdAt": "Timestamp"
}
```

**IMPORTANT:** The field is `entry_type` (snake_case), NOT `entryType`. And `is_visible`, NOT `isVisible`. The web app must use these exact field names.

### Entry Types and Their Colors/Icons:

| entry_type | Icon | Color | Use For |
|---|---|---|---|
| `milestone` | flag.fill | green (#10B981) | Major project milestones |
| `income` | arrow.down.circle.fill | green (#22C55E) | Money received |
| `expense` | arrow.up.circle.fill | red (#EF4444) | Major expenses |
| `allocation` | arrow.triangle.swap | purple (#8B5CF6) | Budget allocations |
| `research` | magnifyingglass | blue (#3B82F6) | Research activities |
| `feature` | star.fill | cyan (#06B6D4) | New features built |
| `bug-fix` | wrench.fill | amber (#F59E0B) | Bug fixes |
| `meeting` | person.3.fill | indigo (#6366F1) | Meetings (including with Sari) |
| other | circle.fill | gray | Anything else |

### What to Put in Achievements:

1. **Milestones** — "Prototype completed", "First deployment", "User testing done"
2. **Income events** — "PSUT grant installment received", with `amount` and `transaction_id` linked
3. **Major expenses** — "Equipment purchased", with `amount` and `transaction_id` linked
4. **Research** — "Literature review completed", "User survey results analyzed"
5. **Features** — "Dashboard v2 launched", "Receipt OCR added"
6. **Bug fixes** — "Fixed currency conversion bug", "Resolved sync issue"
7. **Meetings** — "Meeting with Sari Awwad", "Stakeholder review"
8. **Allocations** — "Q1 budget allocated", "PSUT funds distributed"

### Rules:
- Only entries with `is_visible != false` are shown
- Entries are sorted by `date` descending (most recent first)
- If `imageUrl` is set, it's loaded from Firebase Storage
- `amount` is shown with currency symbol if non-zero
- `summary` takes priority over `description` for display

---

## 11. CSV EXPORT FORMAT (Match These Exactly)

### Period CSV:

```
Period,<period name>
Range,<start date> - <end date>
Status,<Planned|Active|Completed>
Fund,<fund name if set>
Planned Total (JOD),<plannedTotalResolved>
Actual Expenses (JOD),<actualExpense>
Variance (JOD),<plannedTotalResolved - actualExpense>

Date,Type,Category,Vendor,Funding Source,Amount (JOD),Currency,Original Amount,Note
<d MMM yyyy>,<Type>,<resolvedCategory>,<vendor>,<fundingSource>,<amountJOD>,<currency>,<original amount>,<note>
...

Category Summary
Category,Planned (JOD),Actual (JOD),Variance (JOD)
<category>,<planned>,<actual>,<planned - actual>
...
```

### Project CSV:

```
Fund,<fund name>
Grant Total (JOD),<fundTotal>
Total Spent (JOD),<totalSpent>
Remaining (JOD),<fundTotal - totalSpent>
Total Income (JOD),<totalIncome>

Monthly Summary
Month,Income (JOD),Expenses (JOD),Net (JOD),Transactions
<MMMM yyyy>,<income>,<expense>,<net>,<count>
...

All Transactions
Date,Type,Category,Vendor,Amount (JOD),Currency,Original Amount,Note
<d MMM yyyy>,<Type>,<resolvedCategory>,<vendor>,<amountJOD>,<currency>,<original>,<note>
...
```

### Monthly CSV:

```
Monthly Report

Month,Income (JOD),Expenses (JOD),Net (JOD),Transactions
<MMMM yyyy>,<income>,<expense>,<net>,<count>
...
```

---

## 12. COMMON MISTAKES THE WEB IS PROBABLY MAKING

1. **Classifying `payment` as expense** — `payment` is INCOME (money received). Only `bill` and `cost` are expenses.
2. **Not converting USD to JOD** — All sums must use `amountJOD`, not raw `amount`.
3. **Clamping negative cash to zero** — `inCash = psutReceived - psutSpent` can and SHOULD go negative.
4. **Using `periodId` instead of date range** — Transactions are assigned to periods primarily by DATE, not by `periodId`.
5. **Not computing carry-over** — Each period's balance carries into the next. This is cumulative.
6. **Using `spentAmount` field from Firestore** — This is a denormalized field that may be stale. Always recompute: `SUM of amountJOD where isExpense`.
7. **Mixing `totalSpent` with `psutSpent`** — `totalSpent` = ALL expenses. `psutSpent` = only expenses where `fundingSource == "psut"`.
8. **Not handling `allocatedFunds`** — This is the web's equivalent of `plannedTotal`. If `plannedItems` and `plannedTotal` are both missing, fall back to `allocatedFunds`.
9. **Using wrong collection names** — Must use `ady_` prefix: `ady_transactions`, `ady_periods`, `ady_meetings`, `ady_summaries`.
10. **Using camelCase for timeline fields** — Must use `entry_type` and `is_visible` (snake_case) in Firestore.

---

## 13. QUICK REFERENCE — All Numbers in One Place

```
// For each transaction:
amountJOD = currency == "USD" ? amount * 0.707 : amount
isExpense = type in ["bill", "cost"]
isIncome = type in ["payment", "check", "revenue"]

// For each period:
actualExpense = SUM(amountJOD WHERE isExpense)
actualIncome = SUM(amountJOD WHERE isIncome)
netCash = actualIncome - actualExpense
plannedTotal = plannedItems?.sum ?? plannedTotal ?? allocatedFunds ?? 0
variance = plannedTotal - actualExpense

// Carry-over (chronological):
cumulative = 0
for each period (sorted by startDate):
  cumulative += (actualIncome - actualExpense)
  // cumulative is the carry-out for this period

// PSUT grant:
psutReceived = SUM(amountJOD WHERE isIncome AND fundingSource == "psut")
psutSpent = SUM(amountJOD WHERE isExpense AND fundingSource == "psut")
inCash = psutReceived - psutSpent  // CAN BE NEGATIVE
toReceive = max(0, fundTotal - psutReceived)

// Project totals:
totalSpent = SUM(amountJOD WHERE isExpense)  // ALL funding sources
totalIncome = SUM(amountJOD WHERE isIncome)  // ALL funding sources
isOverspent = totalSpent > totalIncome
```
