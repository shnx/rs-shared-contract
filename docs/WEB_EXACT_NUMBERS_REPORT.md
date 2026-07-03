# Web → iOS: Exact Numbers Report — Please Compare (July 3)

**From:** Web team
**Re:** Here are the exact numbers we compute from Firestore. Please compare with iOS and tell us where we differ. We found several issues.

---

## Our Current Numbers (from Firestore)

### Project Totals
```
Total transactions: 66
Total Spent:    4,897.45 JOD
Total Income:   5,400.00 JOD
PSUT Spent:     4,897.45 JOD
PSUT Received:  5,400.00 JOD
In Cash:        502.55 JOD
To Receive:     4,600.00 JOD
```

### Per-Period Breakdown
```
Period          | Start       | End         | Txns | Income   | Spent    | Net       | Carry In  | Carry Out
----------------|-------------|-------------|------|----------|----------|-----------|-----------|----------
First Payment   | 2025-06-30  | 2025-09-27  | 25   | 2,300.00 | 3,038.78 | -738.78   | 0.00      | -738.78
First payment   | 2025-07-20  | 2025-09-28  | 25   | 2,300.00 | 3,038.78 | -738.78   | -738.78   | -1,477.56
Second payment  | 2025-09-29  | 2025-12-06  | 23   | 1,472.00 | 461.38   | +1,010.62 | -1,477.56 | -466.94
Third payment   | 2025-12-07  | 2026-01-31  | 18   | 1,628.00 | 1,397.29 | +230.71   | -466.94   | -236.23
June 2026       | 2026-05-31  | 2026-06-30  | 2    | 0.00     | 520.00   | -520.00   | -236.23   | -756.23
```

### Income Transactions (3 total — all type "revenue")
| ID | Amount | Currency | Date | Period | Description |
|---|---|---|---|---|---|
| 1764517182099... | 2,300 JOD | JOD | 2025-07-20 | First Payment | "First payment" |
| 1764582913728... | 1,472 JOD | JOD | 2025-10-21 | Second payment | "second payment" |
| 1765989615263... | 1,628 JOD | JOD | 2025-12-17 | Third payment | "fund" |

### Installments
**0 installments found in `ady_installments` collection.** All payments are already in `ady_transactions` as `type: "revenue"`.

---

## Issues We Found

### Issue #1 (CRITICAL): Duplicate periods — "First Payment" counted TWICE

There are TWO overlapping periods for the first payment:
- `CBE91C2E-...` — "First Payment" — Jun 30 to Sep 27, 2025
- `1764518075230...` — "First payment " — Jul 20 to Sep 28, 2025

These overlap almost entirely. All 25 transactions in period 1 also match period 2 by date range. This causes:
- **Double-counting the 2,300 JOD income** (appears in both periods)
- **Double-counting all expenses** in that range
- **Carry-out is wrong** — first period carry is -738.78, second period adds another -738.78 making it -1,477.56

**Question for iOS: Which period is the correct one? Should we delete the duplicate? Or does iOS merge them?**

### Issue #2 (CRITICAL): Transaction types `freelance` and `salary` not classified

3 transactions are neither expense nor income:
| ID | Type | Amount | Description |
|---|---|---|---|
| 1764585236634... | `freelance` | 160 JOD | "builing the website" |
| 1764587327100... | `salary` | 160 JOD | "Website designer" |
| 1767798201065... | `freelance` | 350 JOD | "Apple ios developer to control the app" |

These are clearly **expenses** (paying people for work). But our iOS-aligned code only recognizes `cost` and `bill` as expenses. `freelance` and `salary` are not in the iOS spec.

**Question for iOS: How do you classify `freelance` and `salary` types? Are they expenses? Should we add them to `isExpense`?**

### Issue #3: EUR currency not converted

Some transactions have `currency: "EUR"` but our converter only handles USD→JOD:
- 28.56 EUR — "Cloud Starter subscription" — treated as 28.56 JOD
- 21.99 EUR — "Google Al Pro (2 TB) subscription" — treated as 21.99 JOD
- 99.00 EUR — "Apple Developer Program annual membership" — treated as 99.00 JOD

**Question for iOS: Do you have an EUR→JOD conversion rate? Or should these be treated as USD?**

### Issue #4: Transactions outside period dates but assigned by periodId

The "Third payment" period ends 2026-01-31, but several transactions with `periodId` = "Third payment" have dates in Feb 2026 (after the period ended). Our `transactionInPeriod` uses `dateInRange || periodMatch`, so these are included by periodId match.

**Question for iOS: Should we use date range ONLY (ignore periodId for out-of-range dates)? Or is the current OR logic correct?**

---

## What iOS Shows (from your `PAYMENT_CALCULATION_BUG.md`)

You showed this expected table:
```
Period     | Planned | Received | Spent  | Net    | Carry
June 2026  | 5,000   | 3,000    | 3,200  | -200   | -200
July 2026  | 5,000   | 3,000    | 0      | +3,000 | +2,800
Aug 2026   | 5,000   | 3,000    | 0      | +3,000 | +5,800
```

But our actual Firestore data shows:
- 3 payments of 2,300 / 1,472 / 1,628 JOD (not 3,000 each)
- 5 periods (not 3), with duplicate "First Payment"
- 66 transactions total

**Please send us the exact numbers iOS shows right now so we can compare.**

---

## Summary of Questions

1. **Duplicate "First Payment" period** — which one is correct? `CBE91C2E-...` or `1764518075230...`?
2. **`freelance` and `salary` types** — are these expenses? How does iOS classify them?
3. **EUR currency** — what conversion rate to JOD? Or should these be USD?
4. **Period date range vs periodId** — if a transaction's date is outside the period but periodId matches, should it be included?
5. **What numbers does iOS show right now?** — Please share exact Total Spent, Total Income, In Cash, and per-period breakdown so we can compare.
