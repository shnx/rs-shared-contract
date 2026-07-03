# Web Bugs Found & Fixed — 2nd/3rd Payment + Period Table (July 3)

**From:** iOS team
**Re:** We checked your web code, found the bugs causing wrong 2nd/3rd payment numbers, and fixed them. Also added transaction count per period to the summary table.

---

## Bugs Found in `server/sharedBudget.js`

### Bug #1 (CRITICAL): Case-sensitive `type` matching

**Was:**
```javascript
function isExpense(tx) {
  return tx.type === 'cost' || tx.type === 'bill';
}
function isIncome(tx) {
  return tx.type === 'revenue' || tx.type === 'payment' || tx.type === 'check';
}
```

If Firestore stores `type: "Payment"` or `type: "Cost"` (capitalized), these functions return `false`. The 2nd and 3rd PSUT payments would be **invisible** — not counted as income OR expense.

**Fixed:**
```javascript
function isExpense(tx) {
  if (tx.isExpense != null) return tx.isExpense;
  const t = (tx.type || '').toLowerCase();
  return t === 'cost' || t === 'bill';
}
function isIncome(tx) {
  if (tx.isIncome != null) return tx.isIncome;
  const t = (tx.type || '').toLowerCase();
  return t === 'revenue' || t === 'payment' || t === 'check';
}
```

Also added `isExpense`/`isIncome` boolean flag check (iOS may set these directly on the transaction).

### Bug #2 (CRITICAL): Case-sensitive `fundingSource` matching

**Was:**
```javascript
if (tx.fundingSource === 'psut') psutSpent += amt;
if (tx.fundingSource === 'psut') psutReceived += amt;
```

If `fundingSource` is stored as `"PSUT"`, `"Psut"`, or `"PSUT Venture Lab"`, it won't match. The payments exist but don't count toward the PSUT fund card.

**Fixed:**
```javascript
if ((tx.fundingSource || '').toLowerCase() === 'psut') psutSpent += amt;
if ((tx.fundingSource || '').toLowerCase() === 'psut') psutReceived += amt;
```

### Bug #3 (CRITICAL): `inCash` clamped to zero

**Was:**
```javascript
const inCash = Math.max(0, psutReceived - psutSpent);
```

iOS **intentionally allows negative `inCash`** to show overspend. If you spent more PSUT money than you received, `inCash` should be negative and shown in red. Clamping to zero hides the problem.

**Fixed:**
```javascript
const inCash = psutReceived - psutSpent; // CAN BE NEGATIVE — do NOT clamp
```

### Bug #4: `amountInJOD` missing `amountJOD` field fallback

**Was:**
```javascript
function amountInJOD(tx) {
  const amount = Number(tx.amount) || 0;
  if (tx.currency === 'USD') return amount * 0.707;
  return amount;
}
```

iOS sometimes embeds a pre-calculated `amountJOD` field in shared doc transactions. The server ignored it.

**Fixed:**
```javascript
function amountInJOD(tx) {
  if (tx.amountJOD != null) return Number(tx.amountJOD) || 0;
  const amount = Number(tx.amount) || 0;
  if (tx.currency === 'USD') return amount * 0.707;
  return amount;
}
```

---

## Bugs Found in Dashboard (`SharedBudgetDashboard.tsx`)

### Bug #5: Period table missing transaction count

The period summary table showed Planned, Received, Spent, Net, Carry — but **not how many transactions** are in each period. This makes it hard to spot missing transactions.

**Fixed:** Added a "Txns" column showing the transaction count per period, with a total in the footer row.

---

## Files Changed

| File | Changes |
|---|---|
| `server/sharedBudget.js` | Fixed `isExpense`/`isIncome` (case-insensitive + bool flag), fixed `fundingSource` matching (case-insensitive), fixed `inCash` (no clamp), added `amountJOD` fallback, added `transactionCount` to carry-over |
| `src/utils/sharedBudgetClient.ts` | Added `transactionCount` to carry-over response |
| `src/utils/financials.ts` | Added `transactionCount`, `actualExpense`, `actualIncome` to `CarryOverResult` interface and `computeCarryOver` |
| `src/website/SharedBudgetDashboard.tsx` | Added "Txns" column to period summary table, updated `CarryOver` interface |

---

## What the Period Table Now Shows

```
┌──────────────┬─────────┬──────────┬────────┬─────────┬──────┬─────────┐
│ Period       │ Planned │ Received │ Spent  │ Net     │ Txns │ Carry   │
├──────────────┼─────────┼──────────┼────────┼─────────┼──────┼─────────┤
│ June 2026    │ 5,000   │ 3,000    │ 3,200  │ -200    │ 8    │ -200    │
│ July 2026    │ 5,000   │ 3,000    │ 0      │ +3,000  │ 1    │ +2,800  │
│ Aug 2026     │ 5,000   │ 3,000    │ 0      │ +3,000  │ 1    │ +5,800  │
├──────────────┼─────────┼──────────┼────────┼─────────┼──────┼─────────┤
│ Total        │ 15,000  │ 9,000    │ 3,200  │ +5,800  │ 10   │ +5,800  │
└──────────────┴─────────┴──────────┴────────┴─────────┴──────┴─────────┘
```

The **Txns** column lets you immediately spot if a period is missing transactions (e.g., if July shows 0 txns but should have the 2nd payment, you know something is wrong).

---

## How to Verify the Fix

After deploying, check:
1. The PSUT fund card shows correct `received` (should be sum of all PSUT payments)
2. `inCash` can go negative (red) when overspent
3. The period table shows transaction counts per period
4. The carry-over column matches iOS

If numbers still don't match iOS, check Firestore for the actual `type` and `fundingSource` values — they might have unexpected casing or whitespace.
