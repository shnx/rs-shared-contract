# Web → iOS: Our Exact Calculation Code (July 3)

**From:** Web team
**Re:** Here is every calculation function we use, with exact code. Please compare with iOS and tell us where we differ.

---

## 1. Data Sources

### Collections we read from:
- `ady_transactions` (primary) + `transactions` (unprefixed, legacy) — merged by ID, newest wins
- `ady_periods` (primary) + `periods` (unprefixed) — merged by ID
- `ady_installments` — for PSUT grant payments

### Transaction fetch (`database.ts`):
```typescript
// Read from both ady_transactions and transactions, merge by ID — matches iOS
const [primary, secondary] = await Promise.all([
  firebaseDB.getAll('ady_transactions'),
  firebaseDB.getAll('transactions'),
]);
const merged = mergeByID(primary, secondary); // newest updatedAt wins
```

### Project filter:
```typescript
getByProject: async (projectId) => {
  const all = await getAll();
  const withProjectId = all.filter(t => t.projectId);
  if (withProjectId.length === 0) return all; // fallback: ADY data may not have projectId
  return all.filter(t => t.projectId === projectId);
}
```

---

## 2. Installment Merge (NEW — fixes 2nd/3rd payment bug)

Received installments are merged as synthetic `payment` transactions:

```typescript
const installmentTxns = iData
  .filter(i => i.status === 'received' && i.receivedAmount > 0)
  .filter(i => !txns.some(t => t.installmentId === i.id)) // avoid double-counting
  .map(i => ({
    id: `installment_${i.id}`,
    projectId: i.projectId,
    type: 'payment',           // INCOME
    description: i.title || `Installment #${i.installmentNumber}`,
    amount: i.receivedAmount,
    currency: 'JOD',
    date: i.receivedDate,
    periodId: i.periodId || '',
    installmentId: i.id,
    category: 'Funding',
    fundingSource: 'psut',
    evidenceAttached: false,
    evidenceFiles: [],
  }));

const allTxns = [...txns, ...installmentTxns];
```

**Question: Does iOS create a transaction in `ady_transactions` when an installment is received? If yes, we may be double-counting. If no, this synthetic approach is correct.**

---

## 3. Transaction Classification (`financials.ts`)

```typescript
// cost/bill = expense; revenue/payment/check = income
// Respect isExpense/isIncome boolean flags if present (iOS sets these)

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

---

## 4. Currency Conversion (`currency.ts`)

```typescript
const USD_TO_JOD_RATE = 0.707;

function convertToJOD(amount, currency) {
  if (currency === 'USD') return amount * 0.707;
  return amount; // JOD or anything else passes through
}

function amountJOD(tx) {
  return convertToJOD(tx.amount, tx.currency);
}
```

**For shared report (embedded data):** if iOS pre-converts and includes `amountJOD` field, we use that directly:
```typescript
function amountInJOD(tx) {
  if (tx.amountJOD != null) return Number(tx.amountJOD) || 0;
  const amount = Number(tx.amount) || 0;
  if (tx.currency === 'USD') return amount * 0.707;
  return amount;
}
```

---

## 5. Period Assignment (`financials.ts`)

```typescript
// Date range is PRIMARY, periodId is SECONDARY — both checked with OR
function transactionInPeriod(tx, period) {
  const txDate = new Date(tx.date);
  const start = new Date(period.startDate);
  const end = new Date(period.endDate);
  end.setHours(23, 59, 59, 999); // include entire end day
  const dateInRange = txDate >= start && txDate <= end;
  const periodMatch = tx.periodId != null && tx.periodId === period.id;
  return dateInRange || periodMatch;
}
```

---

## 6. Project-Level Totals (`financials.ts`)

```typescript
function computeProjectFinancials(transactions) {
  let totalSpent = 0, totalIncome = 0;
  let psutSpent = 0, psutReceived = 0;

  for (const tx of transactions) {
    const amt = amountJOD(tx);
    const fs = (tx.fundingSource || '').toLowerCase();
    if (isExpense(tx)) {
      totalSpent += amt;
      if (fs === 'psut') psutSpent += amt;
    } else if (isIncome(tx)) {
      totalIncome += amt;
      if (fs === 'psut') psutReceived += amt;
    }
  }

  const FUND_TOTAL = 10000;
  const inCash = psutReceived - psutSpent;       // CAN BE NEGATIVE — no clamp
  const toReceive = Math.max(0, FUND_TOTAL - psutReceived);
  const receivedPct = FUND_TOTAL > 0 ? (psutReceived / FUND_TOTAL) * 100 : 0;
  const isOverspent = totalSpent > totalIncome;
  const overspendAmount = Math.max(0, totalSpent - totalIncome);

  // Progress bar segments
  const spentSeg = Math.min(psutSpent, FUND_TOTAL);
  const cashSeg = Math.max(0, Math.min(psutReceived, FUND_TOTAL) - spentSeg);
  const pendingSeg = FUND_TOTAL - spentSeg - cashSeg;

  return { totalSpent, totalIncome, psutSpent, psutReceived, fundTotal: FUND_TOTAL,
           inCash, toReceive, receivedPct, isOverspent, overspendAmount,
           spentSeg, cashSeg, pendingSeg, txnCount: transactions.length };
}
```

---

## 7. Period-Level Totals (`financials.ts`)

```typescript
function computePeriodFinancials(period, transactions) {
  const periodTxns = transactions.filter(t => transactionInPeriod(t, period));

  let actualExpense = 0, actualIncome = 0;
  for (const tx of periodTxns) {
    const amt = amountJOD(tx);
    if (isExpense(tx)) actualExpense += amt;
    else if (isIncome(tx)) actualIncome += amt;
  }

  const netCash = actualIncome - actualExpense;

  // Days calculation
  const start = new Date(period.startDate);
  const end = new Date(period.endDate);
  end.setHours(23, 59, 59, 999);
  const daysInPeriod = Math.max(1, Math.ceil((end.getTime() - start.getTime()) / 86400000));
  const avgDailySpend = actualExpense / daysInPeriod;
  const avgMonthlySpend = actualExpense / Math.max(1, daysInPeriod / 30);

  // Planned total resolution (iOS priority):
  // 1. plannedItems sum → 2. plannedTotal → 3. allocatedFunds → 4. 0
  let plannedTotal = 0;
  if (period.plannedItems?.length > 0) {
    plannedTotal = period.plannedItems.reduce((sum, item) => sum + (Number(item.amount) || 0), 0);
  } else if (period.plannedTotal != null) {
    plannedTotal = Number(period.plannedTotal) || 0;
  } else if (period.allocatedFunds != null) {
    plannedTotal = Number(period.allocatedFunds) || 0;
  }

  const variance = plannedTotal - actualExpense;

  return { actualExpense, actualIncome, netCash, daysInPeriod,
           avgDailySpend, avgMonthlySpend, plannedTotal, variance,
           transactionCount: periodTxns.length };
}
```

---

## 8. Carry-Over Timeline (`financials.ts`)

```typescript
function computeCarryOver(periods, transactions) {
  const sorted = [...periods].sort(
    (a, b) => new Date(a.startDate).getTime() - new Date(b.startDate).getTime()
  );
  let cumulative = 0;

  return sorted.map(p => {
    const pf = computePeriodFinancials(p, transactions);
    const periodNet = pf.netCash;        // income - expense for THIS period
    const carryIn = cumulative;          // what we brought from prior periods
    const carryOut = carryIn + periodNet; // new running balance
    cumulative = carryOut;

    return {
      periodId: p.id,
      periodName: p.name,
      periodNet,
      carryIn,
      carryOut,
      isDebt: carryOut < 0,
    };
  });
}
```

---

## 9. Unassigned Transactions

Transactions that don't fall in ANY period — shown as warning on Budget page:

```typescript
const unassigned = allTxns.filter(t =>
  !periods.some(p => transactionInPeriod(t, p))
);
// These count in project-level totals but not in any period
```

---

## 10. Shared Report Page (`sharedBudgetClient.ts`)

For the `/shared/:token` page, when data is embedded from iOS:

1. **Fund & summary**: Recalculated from embedded `transactions` array — does NOT trust iOS pre-calculated `fund` or `summary`
2. **Periods**: Recalculated from embedded transactions using `computePeriodTotals`
3. **Carry-over**: Recalculated from embedded transactions using `computeCarryOver`
4. **Achievements**: Stripped to display-only (id, title, summary, date, entryType, imageUrl) — NO amount/currency/category
5. **amountJOD**: If iOS embeds pre-converted `amountJOD` field, we use it directly

---

## 11. Date Handling

```typescript
// database.ts — convertDate handles Firestore Timestamps
function convertDate(val) {
  if (!val) return new Date();       // ← WARNING: returns current date for null
  if (val instanceof Date) return val;
  if (typeof val === 'string') return new Date(val);
  if (typeof val === 'object') {
    if (val.seconds !== undefined)
      return new Date(val.seconds * 1000 + (val.nanoseconds || 0) / 1e6);
    if (val._seconds !== undefined)
      return new Date(val._seconds * 1000 + (val._nanoseconds || 0) / 1e6);
  }
  return new Date(val);
}
```

**Potential issue**: If `date` is null/missing, `convertDate` returns `new Date()` (today). This could assign a transaction to the wrong period. Should we return `null` instead and exclude the transaction?

---

## Summary: Please Verify

Compare each function above with your iOS Swift code and tell us:

1. **Installment merge** — Do you create a transaction in `ady_transactions` when an installment is received? Or only `ady_installments`?
2. **`fundingSource` values** — Are they always lowercase `"psut"`? Or could they be `"PSUT"` or `"Psut"`?
3. **`currency` field** — Is it always set? What happens if it's null?
4. **`amount` field** — Is it always a number? Could it be a string?
5. **Date format** — How are dates stored in Firestore? Timestamps with seconds/nanoseconds?
6. **`convertDate` null fallback** — Should transactions with null dates be excluded?
7. **Progress bar segments** — Do our `spentSeg`/`cashSeg`/`pendingSeg` formulas match iOS?
8. **Planned total** — Do you use `plannedItems` or `plannedTotal` or `allocatedFunds`?
9. **Unassigned transactions** — Do you show them? Do they count in project totals?
10. **Period summary table** — Our columns are: Period, Planned, Received, Spent, Net, Carry. Does iOS show the same columns?

Please point out ANY difference. We will fix it immediately.
