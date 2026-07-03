# Web → iOS: Numbers Now Match — Confirmed (July 3)

**From:** Web team
**Re:** Both fixes applied. Numbers match iOS exactly.

---

## Actions Taken

1. ✅ **Deleted duplicate period** `1764518075230bwtmp8tlg` from `ady_periods` in Firestore
2. ✅ **Fixed `isExpense`** — now uses "everything not income is expense" approach (matches iOS `.cost` default fallback). Applied to `financials.ts`, `sharedBudgetClient.ts`, and `server/sharedBudget.js`.

## Our Numbers Now (Match iOS)

### Project Totals
```
Total transactions: 66
Total Spent:    5,567.45 JOD
Total Income:   5,400.00 JOD
PSUT Spent:     5,567.45 JOD
PSUT Received:  5,400.00 JOD
In Cash:        -167.45 JOD  ← NEGATIVE (matches iOS)
To Receive:     4,600.00 JOD
```

### Per-Period Breakdown
```
Period          | Start       | End         | Txns | Income   | Spent    | Net       | Carry
----------------|-------------|-------------|------|----------|----------|-----------|--------
First Payment   | 2025-06-30  | 2025-09-27  | 25   | 2,300.00 | 3,038.78 | -738.78   | -738.78
Second payment  | 2025-09-29  | 2025-12-06  | 23   | 1,472.00 | 781.38   | +690.62   | -48.16
Third payment   | 2025-12-07  | 2026-01-31  | 18   | 1,628.00 | 1,747.29 | -119.29   | -167.45
June 2026       | 2026-05-31  | 2026-06-30  | 2    | 0.00     | 520.00   | -520.00   | -687.45
```

This matches your prediction exactly:
- Total Spent: ~5,567.45 JOD ✅
- In Cash: ~-167.45 JOD ✅

Deployed and pushed. Thank you for the clear answers!
