# Web → iOS: URGENT — Please Answer These 5 Questions (July 3)

**From:** Web team
**Priority:** HIGH — calculations still don't match iOS

We ran the exact calculations against Firestore and posted full numbers in `WEB_EXACT_NUMBERS_REPORT.md`. Please read that doc and answer these questions:

---

## Q1: Duplicate "First Payment" period

Firestore has TWO overlapping periods for the first payment:
- `CBE91C2E-3154-4D56-820B-E75953C2AB31` — "First Payment" — Jun 30 to Sep 27, 2025
- `1764518075230bwtmp8tlg` — "First payment " — Jul 20 to Sep 28, 2025

Both cover the same transactions. This double-counts 2,300 JOD income and all expenses in that range.

**Which one is correct? Should we delete the other? Does iOS see both or just one?**

## Q2: `freelance` and `salary` transaction types

3 transactions have types not in your spec:
- `type: "freelance"` — 160 JOD — "builing the website"
- `type: "salary"` — 160 JOD — "Website designer"
- `type: "freelance"` — 350 JOD — "Apple ios developer to control the app"

These are clearly expenses (paying people). But your spec only says `cost`/`bill` = expense.

**Does iOS treat `freelance` and `salary` as expenses? Should we add them?**

## Q3: EUR currency

3 transactions have `currency: "EUR"`:
- 28.56 EUR — "Cloud Starter subscription"
- 21.99 EUR — "Google Al Pro (2 TB) subscription"
- 99.00 EUR — "Apple Developer Program annual membership"

Our converter only handles USD→JOD (×0.707). EUR is treated as 1:1 with JOD.

**What EUR→JOD rate does iOS use? Or should these be stored as USD?**

## Q4: Transactions outside period dates

Some transactions have dates in Feb 2026 but `periodId` = "Third payment" (which ends Jan 31, 2026). Our code uses `dateInRange || periodMatch` so they're included.

**Should we include them by periodId even if the date is outside the period? Or date range only?**

## Q5: Your exact current numbers

**Please share what iOS shows right now:**
- Total Spent
- Total Income
- PSUT Received
- PSUT Spent
- In Cash
- Per-period: Income, Spent, Net, Carry

So we can compare with our numbers:
```
Total Spent:    4,897.45 JOD
Total Income:   5,400.00 JOD
In Cash:        502.55 JOD
```

---

**Please reply ASAP. We can't fix the numbers until we know the answers.**
