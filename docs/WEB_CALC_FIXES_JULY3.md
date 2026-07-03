# Web → iOS: Budget Calculation Fixes + Portal Improvements Response (July 3)

**From:** Web team
**Re:** Fixes based on `IOS_CALCULATION_CODE_REFERENCE.md` + response to `PORTAL_IMPROVEMENT_IDEAS.md`

---

## All 6 Calculation Bugs FIXED

1. **Carry-over** — cumulative net flows forward, matches `carryOverTimeline()`
2. **In Cash negative** — removed `Math.max(0,...)`, shows red
3. **Payment = income** — only `cost`/`bill` = expense, `revenue`/`payment`/`check` = income
4. **Date range PRIMARY** — `transactionInPeriod` uses `||` not early return on `periodId`
5. **No cached fields** — always recompute from transactions, no `periodDB.update()`
6. **Unassigned transactions** — amber warning shows count + spent/income totals

## Shared Report Page Also Fixed

- Recalculates fund/summary/periods/carryOver from embedded transactions only
- Achievements stripped to display-only (no amount/currency)
- Case-insensitive `fundingSource`
- Fallback path fetches ALL docs then filters by projectId

## Portal Improvements — Agreed

Will build: Spending Forecast, Cash Flow chart, Meeting notes CRUD, Achievements transaction linkage, localStorage→Firestore migration

## Questions

1. Auto-create timeline entries: only for `fundingSource == "psut"` or all transactions?
2. Riad Shannak: ready to build CRUD UI, confirm building schema first?
3. Should unassigned transactions be auto-assigned to nearest period or left as-is?
