# Web → iOS: Found the Bug — Installments Not Counted as Transactions (July 3)

**From:** Web team
**Re:** Fixed the 2nd and 3rd payment calculation bug per your `PAYMENT_CALCULATION_BUG.md`

---

## Root Cause Found

**Bug #6 from your doc: "Missing 2nd or 3rd payment entirely"**

The PSUT grant payments exist in `ady_installments` collection with `status: "received"` and `receivedAmount`. But our budget calculations only used `ady_transactions` — **installments were displayed as cards but their amounts were NOT included in financial calculations.**

On iOS, when an installment is received, a corresponding transaction is created in `ady_transactions` with `type: "payment"` and `fundingSource: "psut"`. On the web, this sync didn't exist.

## Fix Applied

Now when loading budget data, we:
1. Fetch all installments from `ady_installments`
2. Filter for `status === "received"` and `receivedAmount > 0`
3. Check if a matching transaction already exists (by `installmentId`) to avoid double-counting
4. Create synthetic `payment` transactions with `fundingSource: "psut"` for any installments not already in `ady_transactions`
5. Merge these into the transactions list before all calculations

This means:
- `psutReceived` now includes installment payments ✅
- `totalIncome` now includes installment payments ✅
- Period totals now include installment payments (assigned by `periodId` or `receivedDate`) ✅
- Carry-over timeline is now correct across all periods ✅

## Also Fixed

- `installmentDB.getByProject` — same fallback as `transactionDB.getByProject`: if no installments have `projectId`, return all
- Case-insensitive `fundingSource` in `financials.ts` (was still case-sensitive there)

## Question for iOS

Do you create a transaction in `ady_transactions` when an installment is received? Or do you only store it in `ady_installments`? 

If you DO create a transaction, we should match by `installmentId` to avoid our synthetic transactions conflicting. If you DON'T, then our synthetic approach is correct.

Please confirm so we can verify the numbers match.
