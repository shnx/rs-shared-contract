# iOS → Web: Budget Tab Fix + Unified Timeline Vision (July 3)

**From:** iOS team
**Re:** How to fix the budget tab, and the plan to merge achievements + transactions into one unified period-organized timeline (shareable, no duplication)

---

## Part 1: How to Fix the Budget Tab

The web budget tab has wrong numbers. Here's exactly how we built it on iOS and what the web needs to match.

### 1.1 The Budget Tab Structure (What It Should Look Like)

```
┌─────────────────────────────────────────────┐
│  PSUT GRANT CARD                            │
│  Fund Name: PSUT Venture Lab                │
│  Total: JD 10,000                           │
│  [Progress bar: spent% / cash% / pending%]  │
│  Spent: JD 3,200   In Cash: JD 1,800       │
│  To Receive: JD 5,000                       │
│  (In Cash shows RED if negative)            │
├─────────────────────────────────────────────┤
│  [Periods] [By Month] [Transactions]  ← tabs│
├─────────────────────────────────────────────┤
│  PERIODS SUMMARY TABLE                      │
│  ┌────────┬────────┬─────────┬───────┬──────┬───────┐
│  │Period  │Planned │Received │Spent  │Net   │Carry  │
│  │Jun 2026│JD 5000 │JD 5000  │JD 3200│+1800 │+1800  │
│  │Jul 2026│JD 5000 │JD 0     │JD 500 │-500  │+1300  │
│  │TOTAL   │JD 10000│JD 5000  │JD 3700│+1300 │+1300  │
│  └────────┴────────┴─────────┴───────┴──────┴───────┘
│  (Carry column = cumulative net across periods)
│  (Red when negative = debt)
├─────────────────────────────────────────────┤
│  PERIOD CARDS (tap to see detail)           │
│  ┌───────────────────────────────────────┐  │
│  │ June 2026  [Active]                  │  │
│  │ 1 Jun – 30 Jun                       │  │
│  │ Planned: JD 5,000  Actual: JD 3,200  │  │
│  │ 8 transactions  →                    │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │ July 2026  [Planned]                 │  │
│  │ 1 Jul – 31 Jul                       │  │
│  │ Planned: JD 5,000  Actual: JD 500    │  │
│  │ 2 transactions  →                    │  │
│  └───────────────────────────────────────┘  │
├─────────────────────────────────────────────┤
│  UNASSIGNED TRANSACTIONS (if any)           │
│  • 3 Jul  Cost  -JD 50.00  Cables          │
│  (These don't fall in any period by date    │
│   range or periodId — they need attention)  │
└─────────────────────────────────────────────┘
```

### 1.2 The Period Summary Table — THE Key Component

This is the table that shows ALL periods with carry-over. **This is what the web is getting wrong** — each period is calculated in isolation instead of cumulatively.

**Our Swift code (BudgetPeriodsView.swift lines 1105-1213):**

```swift
struct PeriodSummaryTable: View {
    let periods: [Period]
    let viewModel: BudgetViewModel

    private var rows: [BudgetViewModel.PeriodCarryOver] {
        viewModel.carryOverTimeline()
    }

    // Totals across all periods
    private var totalPlanned: Decimal { periods.reduce(0) { $0 + $1.plannedTotalResolved } }
    private var totalIncome: Decimal { periods.reduce(0) { $0 + viewModel.actualIncome(in: $1) } }
    private var totalSpent: Decimal { periods.reduce(0) { $0 + viewModel.actualExpense(in: $1) } }
    private var totalNet: Decimal { totalIncome - totalSpent }

    // Each row shows:
    // - Period name
    // - Planned (plannedTotalResolved)
    // - Received (actualIncome in period)
    // - Spent (actualExpense in period)
    // - Net (periodNet = income - expense)
    // - Carry (carryOut = cumulative net across ALL periods up to this one)
}
```

**The carry-over calculation (BudgetPeriodsView.swift lines 549-559):**

```swift
func carryOverTimeline() -> [PeriodCarryOver] {
    let sorted = periods.sorted { $0.startDate < $1.startDate }
    var cumulative: Decimal = 0
    return sorted.map { p in
        let net = netCash(in: p)    // income - expense for THIS period
        let carryIn = cumulative     // what we brought from prior periods
        cumulative += net            // add this period's net
        return PeriodCarryOver(
            period: p,
            periodNet: net,
            carryIn: carryIn,
            carryOut: cumulative     // running balance after this period
        )
    }
}
```

**JavaScript for the web:**

```javascript
function carryOverTimeline(periods, transactions) {
  const sorted = [...periods].sort((a, b) => new Date(a.startDate) - new Date(b.startDate));
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
      planned: plannedTotalResolved(p),
      received: income,
      spent: expense,
      net: net,
      carryIn: carryIn,
      carryOut: cumulative,
      isDebt: cumulative < 0
    };
  });
}

function plannedTotalResolved(period) {
  if (period.plannedItems && period.plannedItems.length > 0) {
    return period.plannedItems.reduce((s, item) => s + item.amount, 0);
  }
  return period.plannedTotal || period.allocatedFunds || 0;
}
```

### 1.3 The Fund Card — In Cash Can Be Negative

```javascript
// CORRECT — let it go negative
const inCash = psutReceived - psutSpent;
const inCashColor = inCash >= 0 ? 'green' : 'red';

// WRONG — don't do this
// const inCash = Math.max(0, psutReceived - psutSpent);
```

### 1.4 The "By Month" View

Groups transactions by month (yyyy-MM), showing income/expense/net per month. This is independent of periods — it's just calendar months.

```javascript
function monthlyBuckets(transactions) {
  const buckets = {};
  for (const t of transactions) {
    const key = monthKey(t.date); // "2026-06"
    if (!buckets[key]) {
      buckets[key] = {
        id: key,
        title: monthTitle(t.date), // "June 2026"
        income: 0,
        expense: 0,
        count: 0
      };
    }
    if (isExpense(t)) buckets[key].expense += amountJOD(t);
    if (isIncome(t)) buckets[key].income += amountJOD(t);
    buckets[key].count++;
  }
  return Object.values(buckets).sort((a, b) => b.id.localeCompare(a.id));
}
```

### 1.5 The "Transactions" View (Flat List)

Shows ALL transactions with a summary header. Each row shows date, type, amount, category, evidence badge, period name.

### 1.6 Period Detail (When You Tap a Period)

Shows:
- Period header (name, date range, status)
- Planned vs Actual by category (table with variance)
- Cash flow timeline (running balance within the period)
- Transaction list for this period
- Average daily/monthly spend
- Spending forecast (if planned period)

---

## Part 2: The Unified Timeline Vision

### The Problem Right Now

We have TWO separate things that overlap:

1. **Achievements** (`timeline_entries`) — 67 entries. Visual timeline with images, titles, descriptions. Some have `transaction_id` linking to a transaction. Some don't.

2. **Transactions** (`ady_transactions`) — 66 entries. Financial records with amounts, evidence, vendors. Some have a linked timeline entry. Some don't.

**The issues:**
- Many achievements ARE transactions (income/expense entries with `transaction_id`) — but the web shows them as separate things with potentially different values
- Some transactions have NO timeline entry — they're "invisible" in the achievements view
- Some timeline entries have NO transaction — they're milestones, meetings, research (non-financial)
- The web developer thinks achievements and transactions are completely different things — they're NOT, they're two views of the same project history

### The Solution: Unified Period-Organized Timeline

**One timeline that shows EVERYTHING — achievements AND transactions — organized by period, with no duplication.**

### How It Works

```
UNIFIED TIMELINE (shareable)
├── Period: June 2026
│   ├── ● 15 Jun  💰 Income  +JD 2,000
│   │   ┃ PSUT Grant Installment Received
│   │   ┃ [from transaction: payment, vendor: PSUT, funding: psut]
│   │   ┃ 📎 View Receipt
│   │   ┃ [has timeline entry with image + summary]
│   │
│   ├── ● 10 Jun  🚩 Milestone
│   │   ┃ Prototype v2 completed
│   │   ┃ [timeline entry only — no transaction]
│   │   ┃ [image]
│   │
│   ├── ● 7 Jun  💸 Expense  -JD 120.00
│   │   ┃ Electricity bill
│   │   ┃ [from transaction: bill, vendor: JEPCO]
│   │   ┃ 📎 View Receipt
│   │   ┃ [NO timeline entry — auto-generated from transaction]
│   │
│   ├── ● 5 Jun  💸 Expense  -JD 500.00
│   │   ┃ Equipment purchased (motors, controllers)
│   │   ┃ [from transaction: cost, vendor: RobotShop]
│   │   ┃ 📎 View Receipt
│   │   ┃ [has timeline entry with image + summary]
│   │
│   └── ● 1 Jun  📅 Meeting
│       ┃ Kickoff meeting with Sari Awwad
│       ┃ [timeline entry only — meeting type]
│
├── Period: July 2026
│   ├── ● 3 Jul  💸 Expense  -JD 50.00
│   │   ┃ Cables and connectors
│   │   ┃ [from transaction: cost, no timeline entry]
│   │   ┃ 📎 View Receipt
│   │
│   └── ● 1 Jul  🚩 Milestone
│       ┃ System integration testing
│       ┃ [timeline entry only]
│
├── Unassigned (no period)
│   ├── ● 20 Jul  💸 Expense  -JD 75.00
│   │   ┃ Miscellaneous supplies
│   │   ┃ [transaction with no period — date outside all periods]
│   │
│   └── (warning: assign to a period)
│
└── Period Summary
    ├── June: Income +JD 5,000  Expense -JD 3,200  Net +JD 1,800
    ├── July: Income +JD 0     Expense -JD 50    Net -JD 50
    └── Total:  Income +JD 5,000  Expense -JD 3,250  Net +JD 1,750
```

### The Merge Logic (No Duplication)

For each date in the project history, we need to show ONE entry. Here's how:

```javascript
function buildUnifiedTimeline(timelineEntries, transactions, periods) {
  const items = [];

  // 1. Add all timeline entries
  for (const entry of timelineEntries) {
    if (entry.transaction_id) {
      // LINKED: timeline entry has a transaction
      const tx = transactions.find(t => t.id === entry.transaction_id);
      if (tx) {
        items.push({
          id: `entry_${entry.id}`,
          date: entry.date || tx.date,
          title: entry.title,
          summary: entry.summary || entry.description,
          imageUrl: entry.imageUrl,
          entryType: entry.entry_type,
          category: entry.category,
          // Financial data FROM THE TRANSACTION (not from entry)
          amount: amountJOD(tx),
          isIncome: isIncome(tx),
          isExpense: isExpense(tx),
          vendor: tx.vendor,
          fundingSource: tx.fundingSource,
          hasEvidence: (tx.evidenceFiles || []).length > 0,
          evidenceFiles: tx.evidenceFiles,
          // Mark as both
          source: 'linked',
          timelineEntryId: entry.id,
          transactionId: tx.id,
          // DON'T add the transaction separately — it's already here
        });
        continue;
      }
    }
    // UNLINKED: timeline entry without transaction (milestone, meeting, etc.)
    items.push({
      id: `entry_${entry.id}`,
      date: entry.date,
      title: entry.title,
      summary: entry.summary || entry.description,
      imageUrl: entry.imageUrl,
      entryType: entry.entry_type,
      category: entry.category,
      amount: entry.amount || null,
      source: 'timeline_only',
      timelineEntryId: entry.id,
      transactionId: null,
    });
  }

  // 2. Add transactions that have NO timeline entry
  const linkedTxIds = new Set(
    timelineEntries
      .filter(e => e.transaction_id)
      .map(e => e.transaction_id)
  );
  for (const tx of transactions) {
    if (!linkedTxIds.has(tx.id)) {
      // This transaction has no timeline entry — auto-generate one
      items.push({
        id: `tx_${tx.id}`,
        date: tx.date,
        title: tx.description || tx.category || tx.type,
        summary: tx.note || tx.notes || null,
        imageUrl: null,
        entryType: isIncome(tx) ? 'income' : isExpense(tx) ? 'expense' : 'other',
        category: tx.category,
        amount: amountJOD(tx),
        isIncome: isIncome(tx),
        isExpense: isExpense(tx),
        vendor: tx.vendor,
        fundingSource: tx.fundingSource,
        hasEvidence: (tx.evidenceFiles || []).length > 0,
        evidenceFiles: tx.evidenceFiles,
        source: 'transaction_only',
        timelineEntryId: null,
        transactionId: tx.id,
      });
    }
  }

  // 3. Sort by date (newest first)
  items.sort((a, b) => new Date(b.date) - new Date(a.date));

  // 4. Group by period
  return groupByPeriod(items, periods);
}

function groupByPeriod(items, periods) {
  const sorted = [...periods].sort((a, b) => new Date(a.startDate) - new Date(b.startDate));
  const result = [];

  for (const period of sorted) {
    const periodItems = items.filter(item => isInPeriodByDate(item, period));
    result.push({
      periodId: period.id,
      periodName: period.name,
      periodStart: period.startDate,
      periodEnd: period.endDate,
      status: period.status || 'active',
      items: periodItems,
      income: periodItems.filter(i => i.isIncome).reduce((s, i) => s + i.amount, 0),
      expense: periodItems.filter(i => i.isExpense).reduce((s, i) => s + i.amount, 0),
      net: 0, // calculated below
    });
    result[result.length - 1].net =
      result[result.length - 1].income - result[result.length - 1].expense;
  }

  // Unassigned items (don't fall in any period)
  const assigned = new Set();
  for (const period of sorted) {
    for (const item of items) {
      if (isInPeriodByDate(item, period)) assigned.add(item.id);
    }
  }
  const unassigned = items.filter(item => !assigned.has(item.id));
  if (unassigned.length > 0) {
    result.push({
      periodId: null,
      periodName: 'Unassigned',
      periodStart: null,
      periodEnd: null,
      status: 'warning',
      items: unassigned,
      income: unassigned.filter(i => i.isIncome).reduce((s, i) => s + i.amount, 0),
      expense: unassigned.filter(i => i.isExpense).reduce((s, i) => s + i.amount, 0),
      net: 0,
    });
    result[result.length - 1].net =
      result[result.length - 1].income - result[result.length - 1].expense;
  }

  return result;
}

function isInPeriodByDate(item, period) {
  if (!item.date || !period.startDate || !period.endDate) return false;
  const d = new Date(item.date);
  const start = new Date(period.startDate);
  start.setHours(0, 0, 0, 0);
  const endExclusive = new Date(period.endDate);
  endExclusive.setHours(0, 0, 0, 0);
  endExclusive.setDate(endExclusive.getDate() + 1);
  return d >= start && d < endExclusive;
}
```

### The Three Source Types

| Source | What it means | How to display |
|---|---|---|
| `linked` | Timeline entry WITH a linked transaction | Show story (title, summary, image) + financial data (amount, vendor, evidence) from the transaction |
| `timeline_only` | Timeline entry WITHOUT a transaction (milestone, meeting, research) | Show story only — no amount, no evidence |
| `transaction_only` | Transaction WITHOUT a timeline entry | Auto-generate display from transaction fields (description as title, note as summary, type as entryType) |

### The Golden Rule: NO DUPLICATION

- If a timeline entry has `transaction_id`, the transaction's data is shown IN the timeline entry. The transaction is NOT shown separately.
- If a transaction has no timeline entry, it appears as an auto-generated entry in the timeline.
- The same financial event never appears twice.

---

## Part 3: Making the Unified Timeline Shareable

### What We Want

A shareable link that shows the unified timeline — organized by periods, with all transactions and achievements, evidence access, and period summaries.

### How It Would Work

1. iOS creates a `sharedViews` document with `reportTemplate: "unified_timeline"`
2. The document contains:
   - All timeline entries (with `transaction_id` links)
   - All transactions (with evidence metadata)
   - All periods (with date ranges)
   - Fund info
3. The web renders it as a vertical timeline grouped by period
4. Each entry shows: date, type icon, title, summary, amount (if financial), evidence button
5. Period headers show: period name, date range, status, income/expense/net summary
6. A "Download as PDF" button at the top

### What the `sharedViews` Document Would Contain

```json
{
  "token": "abc123",
  "reportTemplate": "unified_timeline",
  "projectId": "ady_project_id",
  "createdAt": "2026-07-03T...",
  "revoked": false,
  "fund": {
    "name": "PSUT Venture Lab",
    "total": 10000
  },
  "periods": [
    {
      "id": "period_1",
      "name": "June 2026",
      "startDate": "2026-06-01",
      "endDate": "2026-06-30",
      "status": "active",
      "allocatedFunds": 5000
    }
  ],
  "timelineEntries": [
    {
      "id": "entry_1",
      "title": "PSUT Grant Installment Received",
      "summary": "Second installment of the PSUT Venture Lab grant",
      "date": "2026-06-15",
      "entryType": "income",
      "imageUrl": "gs://...",
      "category": "Grant",
      "transactionId": "tx_001"
    }
  ],
  "transactions": [
    {
      "id": "tx_001",
      "date": "2026-06-15",
      "type": "payment",
      "amount": 2000,
      "currency": "JOD",
      "amountJOD": 2000,
      "description": "PSUT Venture Lab - 2nd installment",
      "vendor": "PSUT Venture Lab",
      "fundingSource": "psut",
      "hasEvidence": true,
      "evidenceCount": 1,
      "isExpense": false,
      "isIncome": true
    }
  ]
}
```

The web would run the `buildUnifiedTimeline()` function (from Part 2) on this data to produce the merged view.

### What the Shared Page Looks Like

```
┌─────────────────────────────────────────────┐
│  ADY Project — Unified Timeline             │
│  [Download as PDF]  [Copy Link]             │
├─────────────────────────────────────────────┤
│  PSUT Venture Lab Fund                      │
│  Total: JD 10,000  Spent: JD 3,200         │
│  In Cash: JD 1,800  To Receive: JD 5,000   │
├─────────────────────────────────────────────┤
│  ▼ JUNE 2026  [Active]                      │
│  Income: +JD 5,000  Expense: -JD 3,200      │
│  Net: +JD 1,800                             │
│                                             │
│  ● 15 Jun  💰 Income  +JD 2,000             │
│    ┃ PSUT Grant Installment Received        │
│    ┃ Vendor: PSUT Venture Lab               │
│    ┃ 📎 View Receipt                        │
│                                             │
│  ● 10 Jun  🚩 Milestone                     │
│    ┃ Prototype v2 completed                 │
│    ┃ [image]                                │
│                                             │
│  ● 7 Jun  💸 Expense  -JD 120.00            │
│    ┃ Electricity bill                       │
│    ┃ Vendor: JEPCO  📎 View Receipt         │
│                                             │
│  ● 1 Jun  📅 Meeting                        │
│    ┃ Kickoff meeting with Sari Awwad        │
│                                             │
├─────────────────────────────────────────────┤
│  ▼ JULY 2026  [Planned]                     │
│  Income: +JD 0  Expense: -JD 50  Net: -50   │
│                                             │
│  ● 3 Jul  💸 Expense  -JD 50.00             │
│    ┃ Cables and connectors                  │
│    ┃ Vendor: Electronix  📎 View Receipt    │
│                                             │
├─────────────────────────────────────────────┤
│  ⚠ UNASSIGNED (1 transaction)               │
│  ● 20 Jul  💸 Expense  -JD 75.00            │
│    ┃ Miscellaneous supplies                 │
│    ┃ ⚠ Not in any period — needs assignment │
├─────────────────────────────────────────────┤
│  TOTALS                                     │
│  Income: +JD 5,000  Expense: -JD 3,345      │
│  Net: +JD 1,655                             │
│  10 entries (6 linked, 2 timeline-only,     │
│  2 transaction-only)                        │
└─────────────────────────────────────────────┘
```

---

## Part 4: What the Web Developer Needs to Understand

### Achievements ARE NOT Separate from Transactions

The web developer currently treats `timeline_entries` and `ady_transactions` as completely different things. They're NOT.

**Think of it this way:**
- A **transaction** is the financial record (amount, vendor, evidence, type)
- A **timeline entry** is the story wrapper (title, summary, image, narrative)
- When `transaction_id` is set, the timeline entry is telling the STORY of that transaction
- When `transaction_id` is null, the timeline entry is a non-financial event (milestone, meeting)

**The web should NOT show two separate lists with different numbers.** It should show ONE unified timeline where:
- Financial events show both the story AND the financial data (from the transaction)
- Non-financial events show just the story
- Transactions without a story still appear (auto-generated entry)

### Why Numbers Don't Match

If the web shows:
- Achievements tab: "Income: JD 4,000" (summing `entry.amount` where `entry_type == "income"`)
- Transactions tab: "Income: JD 5,000" (summing `transaction.amountJOD` where `isIncome`)

**These don't match because:**
1. Some income transactions have NO timeline entry (JD 1,000 missing from achievements)
2. Some timeline entries have `amount` that doesn't match the linked transaction's `amountJOD`
3. Some timeline entries use `currency: "USD"` but the transaction uses `currency: "JOD"`

**The fix:** Always use the transaction's `amountJOD` for financial calculations. The timeline entry's `amount` field is a fallback ONLY when there's no linked transaction.

### What to Do Right Now

1. **Fix the budget tab** — implement the carry-over timeline (Part 1)
2. **Fix the achievements tab** — cross-reference with transactions using `transaction_id` (already spec'd in `ACHIEVEMENTS_TRANSACTION_LINKAGE.md`)
3. **Build the unified timeline view** — use the `buildUnifiedTimeline()` function from Part 2
4. **Build the shareable unified timeline** — `reportTemplate: "unified_timeline"` (Part 3)

### What iOS Will Do

1. Update `buildShareDocData` to support `reportTemplate: "unified_timeline"`
2. Include all timeline entries AND all transactions in the share doc
3. Add a "Share Timeline" button in the Achievements tab
4. The web runs the merge logic on the shared page

---

## Part 5: Database Understanding — Read Carefully

### Collections We Use

| Collection | What's in it | Who writes | Who reads |
|---|---|---|---|
| `timeline_entries` | Achievement/timeline entries (story) | Web + iOS | Both |
| `ady_transactions` | Financial transactions (money) | Web + iOS | Both |
| `ady_periods` | Budget periods (date ranges) | Web + iOS | Both |
| `ady_meetings` | Meeting notes | iOS (web soon) | Both |
| `sharedViews` | Shareable link data | iOS only | Web reads |

### The `transaction_id` Field

This is the ONLY link between a timeline entry and a transaction. It's stored as `transaction_id` (snake_case) in Firestore on the `timeline_entries` document.

```
timeline_entries/{entryId}
  ├── title: "PSUT Grant Received"
  ├── entry_type: "income"
  ├── transaction_id: "abc123"     ← THIS IS THE LINK
  └── ...

ady_transactions/{abc123}
  ├── type: "payment"
  ├── amount: 2000
  ├── currency: "JOD"
  └── ...
```

### What Happened Before (Historical Context)

1. Originally, the website had a `timeline` collection for public achievements (photos, milestones)
2. The iOS app started reading from `timeline_entries` (same data, different collection name)
3. We added `transaction_id` to link financial achievements to transactions
4. The web kept treating them as separate — achievements page shows timeline entries, transactions page shows transactions
5. This caused confusion: same financial event appears in both places with potentially different numbers

### What Should Happen Now

1. The achievements page should NOT independently calculate financial totals
2. Financial totals should ALWAYS come from transactions
3. The achievements timeline should pull amount/vendor/evidence from the linked transaction
4. Transactions without a timeline entry should still be visible in the unified timeline
5. The unified timeline is the SINGLE source of truth for "what happened in this project"

---

*Reply by adding a new `.md` file to `shared/docs/` — we'll pull and read it.*
