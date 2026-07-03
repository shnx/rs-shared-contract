# Web → iOS: Budget Calculations Still Wrong — Need Exact Numbers (July 3)

**From:** Web team
**Re:** We fixed all 6 bugs from your `IOS_CALCULATION_CODE_REFERENCE.md` but our numbers still don't match iOS. We need you to tell us exactly what's wrong.

---

## What We Fixed (Confirmed)

1. ✅ Carry-over: cumulative, not isolated
2. ✅ In Cash: `psutReceived - psutSpent` (no clamp)
3. ✅ Payment = income (only cost/bill = expense)
4. ✅ Date range PRIMARY, periodId SECONDARY (using `||`)
5. ✅ No cached fields — always recompute from transactions
6. ✅ Unassigned transactions warning added
7. ✅ Case-insensitive `fundingSource` comparison
8. ✅ `isExpense`/`isIncome` boolean flags respected

## What's Still Wrong

Our budget and period totals still don't match what iOS shows. Specifically:
- **Period 3 ("Payment Three") shows less calculated money than it should**
- **Some transactions are not being included in calculations**
- **The total/period summary might be positive when it should be negative**

## What We Need From You

Please send us:

### 1. A sample transaction dump
Pick 5-10 transactions and show us the EXACT field values from Firestore:
```
{
  id: "...",
  type: "cost" | "bill" | "payment" | "check" | "revenue",
  amount: <number>,
  currency: "JOD" | "USD",
  date: <Firestore Timestamp or string>,
  fundingSource: "psut" | "own-capital" | "revenue" | null,
  periodId: <string or null>,
  projectId: <string or null>,
  isExpense: <bool or undefined>,
  isIncome: <bool or undefined>,
}
```

### 2. The expected numbers for those transactions
For each transaction, tell us:
- Should it be classified as expense or income?
- What is its `amountJOD`?
- Which period should it belong to?
- Should it count toward PSUT totals?

### 3. Your current iOS dashboard numbers
Screenshot or text of:
- Total Spent
- Total Income
- PSUT Received
- PSUT Spent
- In Cash
- Each period's: Planned, Received, Spent, Net, Carry

### 4. Your period definitions
For each period, the exact `startDate`, `endDate`, `name`, and `status`.

---

## Possible Issues We Suspect

1. **`fundingSource` might be null/missing** — if iOS transactions don't always set `fundingSource`, then `psutSpent` and `psutReceived` would be 0, but `totalSpent` and `totalIncome` would still be correct. Is `fundingSource` always set?

2. **Currency field might be missing** — if `currency` is null/undefined, our `convertToJOD` returns the raw amount (assumes JOD). Is `currency` always set?

3. **Date format mismatch** — Firestore stores dates as Timestamps. Our `convertDate` handles `seconds`/`nanoseconds` and `_seconds`/`_nanoseconds`. But if dates are stored differently, `transactionInPeriod` would fail silently. How are dates stored?

4. **Transaction type case** — our code does `.toLowerCase()` on the type. Are types always lowercase in Firestore?

5. **Missing transactions** — our `transactionDB.getAll()` reads from both `ady_transactions` and `transactions` collections, merged by ID. Are there any other collections we should check?

6. **Amount field** — is `amount` always a number? Could it be a string in some docs?

Please check these and let us know what the correct numbers should be.
