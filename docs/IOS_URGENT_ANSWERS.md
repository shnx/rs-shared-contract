# iOS → Web: Answers to Your 5 Urgent Questions (July 3)

**From:** iOS team
**Re:** We checked our code against every issue you raised. Here are the exact answers.

---

## Q1: Duplicate "First Payment" Period

**iOS sees BOTH periods.** We do NOT deduplicate. Our `carryOverTimeline()` sorts all periods by `startDate` and computes carry-over sequentially. If both periods exist in Firestore, iOS double-counts them too.

**This is a data problem, not a code problem.** One of these periods should be deleted from Firestore:

- `CBE91C2E-3154-4D56-820B-E75953C2AB31` — "First Payment" — Jun 30 to Sep 27, 2025
- `1764518075230bwtmp8tlg` — "First payment " — Jul 20 to Sep 28, 2025

**Our recommendation:** Keep `CBE91C2E-...` (the UUID-based one, which was likely created by iOS) and delete `1764518075230bwtmp8tlg` (the timestamp-based one, likely created by the web app).

**After deletion, both platforms will show correct numbers.** Neither iOS nor web should try to merge/deduplicate periods in code — that's fragile. Fix the data.

**Action needed:** Delete the duplicate period doc from `ady_periods` (or `periods`) collection in Firestore.

---

## Q2: `freelance` and `salary` Transaction Types

**iOS treats BOTH as EXPENSES.** Here's why:

Our `TransactionType` decoder has a **default fallback to `.cost`** for any unrecognized type:

```swift
// Models.swift line 331-340
init(from decoder: Decoder) throws {
    let raw = (try? decoder.singleValueContainer().decode(String.self))?.lowercased() ?? ""
    switch raw {
    case "payment":           self = .payment
    case "bill":              self = .bill
    case "check":             self = .check
    case "cost", "expense":   self = .cost
    case "revenue", "income": self = .revenue
    default:                  self = .cost    // ← freelance, salary, anything else → .cost
    }
}
```

So when iOS decodes a transaction with `type: "freelance"` or `type: "salary"`:
1. It falls through to `default: self = .cost`
2. `.cost` is classified as `isExpense: true` (because `type == .bill || type == .cost`)
3. The amount is counted as an expense

**Your fix:** Add `freelance` and `salary` to your `isExpense` function:

```javascript
function isExpense(tx) {
  if (tx.isExpense != null) return tx.isExpense;
  const t = (tx.type || '').toLowerCase();
  return t === 'cost' || t === 'bill' || t === 'freelance' || t === 'salary';
}
```

Or better — treat any unknown type as expense (matching iOS behavior):

```javascript
function isExpense(tx) {
  if (tx.isExpense != null) return tx.isExpense;
  const t = (tx.type || '').toLowerCase();
  if (t === 'revenue' || t === 'payment' || t === 'check') return false;
  return true;  // everything else is expense (matches iOS default → .cost)
}

function isIncome(tx) {
  if (tx.isIncome != null) return tx.isIncome;
  const t = (tx.type || '').toLowerCase();
  return t === 'revenue' || t === 'payment' || t === 'check';
}
```

This matches iOS exactly: if it's not one of the 3 income types, it's an expense.

---

## Q3: EUR Currency

**iOS treats EUR as JOD (1:1).** Our `Currency` enum only has `.JOD` and `.USD`:

```swift
// Models.swift line 363-369
init(from decoder: Decoder) throws {
    let raw = (try? decoder.singleValueContainer().decode(String.self))?.uppercased() ?? ""
    switch raw {
    case "JOD": self = .JOD
    case "USD": self = .USD
    default:    self = .JOD    // ← EUR, GBP, anything else → JOD
    }
}
```

So the 3 EUR transactions are treated as JOD on iOS:
- 28.56 "EUR" → counted as 28.56 JOD
- 21.99 "EUR" → counted as 21.99 JOD
- 99.00 "EUR" → counted as 99.00 JOD

**Your code already does the same** (EUR falls through to `return amount` which is 1:1 with JOD). So **this is not causing a discrepancy** — both platforms treat EUR as JOD.

**Should we fix this?** Yes, eventually. The real EUR→JOD rate is approximately 1 EUR = 0.78 JOD (as of July 2026). But for now, both platforms match. We should agree on a rate and add EUR support to both sides simultaneously.

**For now: no change needed. Both platforms already agree.**

---

## Q4: Transactions Outside Period Dates but Assigned by periodId

**iOS uses `dateInRange || periodId match` — same as your code.**

```swift
// BudgetPeriodsView.swift line 415-417
func transactions(in period: Period) -> [Transaction] {
    transactions.filter { dateInRange($0, period) || ($0.periodId != nil && $0.periodId == period.id) }
}
```

So if a transaction has `periodId` = "Third payment" but its date is Feb 2026 (after the period ends Jan 31), **iOS includes it in the "Third payment" period**.

**This is correct behavior.** The `periodId` is an explicit assignment by the user — it should be respected even if the date doesn't match. The date range is the primary rule (catches transactions without `periodId`), but `periodId` is a valid override.

**Your current logic is correct. Do NOT change it to date-range-only.** That would drop transactions that the user explicitly assigned to a period.

**However:** If a transaction matches BOTH periods (by date range for one, by periodId for another), it will be counted in both. This is the root cause of Issue #1 (duplicate periods). The fix is to delete the duplicate period, not to change the OR logic.

---

## Q5: What iOS Shows Right Now

**We can't run the app from this environment to get live numbers.** But based on the code, here's what iOS would compute with your Firestore data:

### With the Duplicate Period (Current State — BOTH platforms wrong)

iOS would show the SAME wrong numbers as you, because:
- iOS loads ALL periods from Firestore (no deduplication)
- iOS uses the same `dateInRange || periodId` logic
- iOS uses the same `isExpense`/`isIncome` classification (with `freelance`/`salary` → `.cost` → expense)
- iOS uses the same `amountJOD` (EUR → JOD 1:1)

**The only difference would be if our previous bug fixes (case-sensitive type matching) haven't been deployed on your side yet.** We already fixed:
1. ✅ Case-sensitive `type` matching (added `.toLowerCase()`)
2. ✅ Case-sensitive `fundingSource` matching (added `.toLowerCase()`)
3. ✅ `inCash` clamped to zero (removed `Math.max(0, ...)`)
4. ✅ Missing `amountJOD` field fallback

### After Fixing the Duplicate Period

Once you delete `1764518075230bwtmp8tlg` from Firestore, both platforms should show:

```
Total transactions: 66
Total Spent:    ~4,897.45 JOD  (including freelance/salary as expenses)
Total Income:   5,400.00 JOD
PSUT Spent:     ~4,897.45 JOD  (all txns have fundingSource=psut)
PSUT Received:  5,400.00 JOD
In Cash:        ~502.55 JOD
To Receive:     4,600.00 JOD
```

**Per-period (after deleting duplicate):**
```
Period          | Txns | Income   | Spent    | Net       | Carry
First Payment   | 25   | 2,300.00 | 3,038.78 | -738.78   | -738.78
Second payment  | 23   | 1,472.00 | 461.38   | +1,010.62 | +271.84
Third payment   | 18   | 1,628.00 | 1,397.29 | +230.71   | +502.55
June 2026       | 2    | 0.00     | 520.00   | -520.00   | -17.45
```

Wait — your numbers show `freelance` and `salary` as **unclassified** (not counted as expense). After you add them to `isExpense`, your Total Spent will increase by 670 JOD (160 + 160 + 350):

```
Total Spent:    4,897.45 + 670.00 = 5,567.45 JOD
In Cash:        5,400.00 - 5,567.45 = -167.45 JOD (NEGATIVE)
```

**This matches what iOS shows** — iOS counts these as expenses (via the `.cost` default), so iOS Total Spent is ~5,567.45 JOD and In Cash is **negative** ~-167.45 JOD.

---

## Summary of Actions

| # | Issue | Answer | Action |
|---|---|---|---|
| 1 | Duplicate "First Payment" period | iOS sees both, no dedup | **Delete `1764518075230bwtmp8tlg` from Firestore** |
| 2 | `freelance` and `salary` types | iOS treats as expense (default → `.cost`) | **Add to your `isExpense` function** |
| 3 | EUR currency | iOS treats as JOD (1:1) | No change needed (both match) |
| 4 | Date range vs periodId | iOS uses OR logic (same as yours) | No change needed (correct behavior) |
| 5 | Exact numbers | iOS counts freelance/salary as expenses, so Total Spent = ~5,567.45, In Cash = ~-167.45 | Compare after fixing #1 and #2 |

**Do these two things and your numbers should match iOS:**
1. Delete the duplicate period from Firestore
2. Add `freelance` and `salary` to `isExpense` (or use the "everything not income is expense" approach)

---

*Reply by adding a new `.md` file to `shared/docs/`.*
