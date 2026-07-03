# iOS → Web: 2nd & 3rd Payment Calculation Bug (July 3)

**From:** iOS team
**Re:** The 2nd and 3rd PSUT grant payments are calculated wrong on the web. Here are the exact numbers and the exact bugs causing the discrepancy.

---

## The Problem

The PSUT Venture Lab grant is JD 10,000, paid in installments (payments). On iOS, the fund card shows correct numbers. On the web, the 2nd and 3rd payments are wrong — either missing, double-counted, or classified as expense instead of income.

---

## How iOS Calculates PSUT Payments (CORRECT)

### Step 1: Classify the transaction type

A "payment" is **INCOME** (money coming IN from PSUT). It is NOT an expense.

```swift
// BudgetPeriodsView.swift line 38
var isIncome: Bool { type == .payment || type == .check || type == .revenue }
var isExpense: Bool { type == .bill || type == .cost }
```

**`payment` = INCOME.** If you treat `payment` as expense, EVERYTHING is wrong.

### Step 2: Filter by funding source

Only transactions with `fundingSource == "psut"` count toward the PSUT grant.

```swift
// BudgetPeriodsView.swift lines 362-374

// PSUT money SPENT = expense transactions (cost + bill) with fundingSource "psut"
var psutSpent: Decimal {
    transactions
        .filter { $0.isExpense && ($0.fundingSource ?? "").lowercased() == "psut" }
        .reduce(Decimal(0)) { $0 + $1.amountJOD }
}

// PSUT money RECEIVED = income transactions (payment + check + revenue) with fundingSource "psut"
var psutReceived: Decimal {
    transactions
        .filter { $0.isIncome && ($0.fundingSource ?? "").lowercased() == "psut" }
        .reduce(Decimal(0)) { $0 + $1.amountJOD }
}
```

### Step 3: Convert to JOD

```swift
// Models.swift line 380
// 1 USD = 0.707 JOD
func convert(amount: Decimal, to target: Currency) -> Decimal {
    if self == target { return amount }
    switch (self, target) {
    case (.USD, .JOD): return amount * 0.707
    case (.JOD, .USD): return amount / 0.707
    default: return amount
    }
}
```

### Step 4: Calculate fund card numbers

```swift
// ADYProjectView.swift lines 485-493

let total = Decimal(fundTotal)           // 10,000 JOD (hardcoded)
let received = viewModel.psutReceived     // sum of all PSUT income payments
let spent = viewModel.psutSpent           // sum of all PSUT expenses
let inCash = received - spent             // can be NEGATIVE
let toReceive = max(Decimal(0), total - received)  // never negative
```

---

## The Exact Numbers (What iOS Shows)

Assuming the PSUT grant has 3 installments of JD 3,000 each (total JD 9,000 of JD 10,000):

| Payment # | Firestore doc | `type` | `fundingSource` | `amount` | `currency` | `amountJOD` | `date` | `periodId` |
|---|---|---|---|---|---|---|---|---|
| 1st | `ady_transactions/tx_001` | `payment` | `psut` | 3000 | JOD | 3000 | 1 Jun 2026 | `period_1` |
| 2nd | `ady_transactions/tx_002` | `payment` | `psut` | 3000 | JOD | 3000 | 1 Jul 2026 | `period_2` |
| 3rd | `ady_transactions/tx_003` | `payment` | `psut` | 3000 | JOD | 3000 | 1 Aug 2026 | `period_3` |

### iOS Fund Card Shows:

```
PSUT Venture Lab Grant
Total:     JD 10,000
Received:  JD 9,000   (3,000 × 3 payments)
Spent:     JD 3,200   (example — sum of PSUT expenses)
In Cash:   JD 5,800   (9,000 - 3,200)
To Receive: JD 1,000  (10,000 - 9,000)
Progress:  ████████████████░░░░  90% received
```

### iOS Period Summary Shows:

```
Period     | Planned | Received | Spent  | Net    | Carry
June 2026  | JD 5000 | JD 3,000 | JD 3,200| -200  | -200
July 2026  | JD 5000 | JD 3,000 | JD 0   | +3000  | +2800
Aug 2026   | JD 5000 | JD 3,000 | JD 0   | +3000  | +5800
TOTAL      | JD 15000| JD 9,000 | JD 3,200| +5800 | +5800
```

---

## What the Web Is Getting Wrong (Likely Bugs)

### Bug #1: `payment` type classified as EXPENSE

**This is the most likely bug.** If the web treats `type: "payment"` as an expense (money going out) instead of income (money coming in), then:

- `psutReceived` = JD 0 (no income found)
- `psutSpent` = JD 9,000 (payments counted as expenses!)
- `inCash` = JD 0 - JD 9,000 = **JD -9,000** (massively wrong)
- `toReceive` = JD 10,000 (nothing received)

**The 2nd and 3rd payments appear as EXPENSES instead of INCOME.**

**Fix:**
```javascript
// WRONG — if you're doing this:
function isExpense(t) {
  return ['bill', 'cost', 'payment'].includes(t.type);  // ← payment is NOT expense!
}

// CORRECT:
function isExpense(t) {
  return t.type === 'bill' || t.type === 'cost';
}

function isIncome(t) {
  return t.type === 'payment' || t.type === 'check' || t.type === 'revenue';
}
```

### Bug #2: `fundingSource` field missing or not matched

If the web doesn't check `fundingSource`, or checks it case-sensitively, or uses a different field name:

```javascript
// WRONG — case-sensitive match:
t.fundingSource === 'PSUT'           // fails if stored as 'psut'
t.fundingSource === 'psut'           // fails if stored as 'PSUT'

// CORRECT — case-insensitive:
(t.fundingSource || '').toLowerCase() === 'psut'
```

**If fundingSource isn't matched, the 2nd and 3rd payments won't count toward `psutReceived`.** They'll be in the general income but not in the PSUT fund card.

### Bug #3: Currency conversion not applied

If a payment is stored in USD but the web doesn't convert:

```javascript
// WRONG:
const received = transactions
  .filter(t => isIncome(t) && isPSUT(t))
  .reduce((s, t) => s + t.amount, 0);    // uses raw amount, no conversion

// CORRECT:
const received = transactions
  .filter(t => isIncome(t) && isPSUT(t))
  .reduce((s, t) => s + amountJOD(t), 0);

function amountJOD(t) {
  if (t.currency === 'USD') return t.amount * 0.707;
  return t.amount;
}
```

**Example:** If the 2nd payment is $4,240 USD (= JD 3,000 at 0.707 rate), and the web shows $4,240 instead of JD 3,000, the total is wrong.

### Bug #4: Period assignment by `periodId` only (ignoring date)

If the 2nd payment has `periodId: null` or `periodId: ""` but its date is 1 Jul 2026, and the web only uses `periodId`:

```javascript
// WRONG — only uses periodId:
function isInPeriod(t, period) {
  return t.periodId === period.id;
}
// 2nd payment with periodId=null → NOT in any period → missing from period totals

// CORRECT — date range OR periodId:
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

### Bug #5: Double-counting payments

If the web shows payments in BOTH the achievements/timeline AND the transactions tab, and sums both:

```
Achievements tab:  "Income: JD 9,000"  (from timeline entries with amount)
Transactions tab:  "Income: JD 9,000"  (from transactions)
Total shown:       JD 18,000           ← WRONG (double-counted)
```

**Fix:** Always source financial amounts from `ady_transactions` only. The timeline entry's `amount` field is a display fallback, NOT a source of truth.

### Bug #6: Missing 2nd or 3rd payment entirely

If the web queries transactions with a filter that excludes them:

```javascript
// WRONG — only fetching period 1:
db.collection('ady_transactions').where('periodId', '==', 'period_1').get()
// 2nd and 3rd payments (in period_2 and period_3) are missing!

// CORRECT — fetch ALL transactions, then filter in code:
db.collection('ady_transactions').get()
// Then filter by date range or periodId in JavaScript
```

---

## How to Debug This on the Web

### Step 1: Log all PSUT transactions

```javascript
const allTx = await db.collection('ady_transactions').get();
const psutTx = allTx.docs
  .map(d => ({ id: d.id, ...d.data() }))
  .filter(t => (t.fundingSource || '').toLowerCase() === 'psut');

console.log('PSUT transactions:', psutTx);
console.log('Count:', psutTx.length);
console.log('Payments (income):', psutTx.filter(t => isIncome(t)));
console.log('Costs (expense):', psutTx.filter(t => isExpense(t)));
```

### Step 2: Verify each payment

```javascript
for (const t of psutTx) {
  console.log({
    id: t.id,
    type: t.type,                    // should be "payment" for installments
    isIncome: isIncome(t),           // should be true
    isExpense: isExpense(t),         // should be false
    amount: t.amount,                // raw amount
    currency: t.currency,            // JOD or USD
    amountJOD: amountJOD(t),         // converted
    fundingSource: t.fundingSource,  // "psut"
    date: t.date,                    // should be in the right period
    periodId: t.periodId             // might be null — that's OK
  });
}
```

### Step 3: Compare with iOS numbers

```javascript
const received = psutTx
  .filter(t => isIncome(t))
  .reduce((s, t) => s + amountJOD(t), 0);

const spent = psutTx
  .filter(t => isExpense(t))
  .reduce((s, t) => s + amountJOD(t), 0);

console.log('PSUT Received:', received);   // should match iOS
console.log('PSUT Spent:', spent);         // should match iOS
console.log('In Cash:', received - spent); // should match iOS
console.log('To Receive:', Math.max(0, 10000 - received));
```

**If `received` is 0 or less than expected, you have Bug #1 or #2.**
**If `received` is in USD amounts, you have Bug #3.**
**If some payments are missing, you have Bug #4 or #6.**
**If the total is double, you have Bug #5.**

---

## The Exact Firestore Documents to Check

Go to Firestore and look at `ady_transactions` collection. Find documents where `fundingSource == "psut"` and `type == "payment"`. You should find 3 (or however many installments have been received).

### What each payment document looks like:

```
ady_transactions/{txId}
  type: "payment"           ← MUST be "payment" (income)
  fundingSource: "psut"     ← MUST be "psut" (case-insensitive)
  amount: 3000              ← raw amount
  currency: "JOD"           ← or "USD" (then amountJOD = amount × 0.707)
  date: July 1, 2026        ← must fall within a period's date range
  periodId: "period_2"      ← might be null — that's OK, date range is primary
  description: "PSUT Venture Lab - 2nd installment"
  vendor: "PSUT Venture Lab"
  evidenceFiles: [...]
```

### What the web should show for each payment:

| Payment | `type` | `isIncome` | `isExpense` | `amount` | `amountJOD` | Shows in fund card as |
|---|---|---|---|---|---|---|
| 1st | `payment` | `true` | `false` | 3000 | 3000 | +JD 3,000 received |
| 2nd | `payment` | `true` | `false` | 3000 | 3000 | +JD 3,000 received |
| 3rd | `payment` | `true` | `false` | 3000 | 3000 | +JD 3,000 received |

**If any of these show `isIncome: false` or `isExpense: true`, THAT'S THE BUG.**

---

## Summary: What to Fix

1. **Check `type` classification** — `payment` is INCOME, not expense. This is the #1 suspect.
2. **Check `fundingSource` matching** — use case-insensitive comparison: `(t.fundingSource || '').toLowerCase() === 'psut'`
3. **Check currency conversion** — always use `amountJOD(t)` not `t.amount` when summing
4. **Check period assignment** — use date range OR `periodId`, not `periodId` alone
5. **Don't double-count** — financial totals come from `ady_transactions` only, never from `timeline_entries`
6. **Fetch ALL transactions** — don't filter by `periodId` in the Firestore query; fetch all and filter in code

---

*Check your code against each bug above. Reply with which one(s) you found.*
