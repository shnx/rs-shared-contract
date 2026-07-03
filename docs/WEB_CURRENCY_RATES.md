# Web → iOS: Currency Rates — Please Match (July 3)

**From:** Web team
**Re:** We added EUR→JOD conversion. Please update iOS to match these exact rates.

---

## Current Rates (Web)

```typescript
const USD_TO_JOD_RATE = 0.707;   // 1 USD = 0.707 JOD
const EUR_TO_JOD_RATE = 0.78;    // 1 EUR = 0.78 JOD
```

## What Changed

Previously, EUR was treated as JOD (1:1) on both platforms. Now the web converts EUR properly:

```typescript
function convertToJOD(amount, currency) {
  if (currency === 'USD') return amount * 0.707;
  if (currency === 'EUR') return amount * 0.78;
  return amount; // JOD
}
```

Applied to:
- `src/utils/currency.ts` (main app)
- `src/utils/sharedBudgetClient.ts` (shared report page)
- `server/sharedBudget.js` (server-side)

## What iOS Needs to Do

1. **Add EUR to your `Currency` enum** (currently only `.JOD` and `.USD`)
2. **Add EUR→JOD conversion** in your `convert(amount:to:)` function:
```swift
case (.EUR, .JOD): return amount * 0.78
case (.JOD, .EUR): return amount / 0.78
```
3. **Update your `Currency` decoder** to recognize "EUR" instead of falling back to `.JOD`

## Impact on Current Numbers

3 EUR transactions affected:
- 28.56 EUR → was 28.56 JOD → now **22.28 JOD** (28.56 × 0.78)
- 21.99 EUR → was 21.99 JOD → now **17.15 JOD** (21.99 × 0.78)
- 99.00 EUR → was 99.00 JOD → now **77.22 JOD** (99.00 × 0.78)

**Total difference: -149.55 EUR was counted as 149.55 JOD, now 116.65 JOD**
**Delta: -32.90 JOD** (Total Spent decreases by 32.90)

### New numbers after EUR fix:
```
Total Spent:    5,567.45 - 32.90 = 5,534.55 JOD
Total Income:   5,400.00 JOD (unchanged)
In Cash:        -167.45 + 32.90 = -134.55 JOD
```

Please confirm:
1. Do you agree with the 0.78 EUR→JOD rate?
2. Will you add EUR support to iOS?
3. Should we use a different rate?

**Once you update iOS, both platforms will match exactly.**
