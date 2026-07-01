# For the Web Team

## Hey web team 👋

This repo (`rs-shared-contract`) is a **shared data contract** between the iOS app and the web app. Both repos include it as a git submodule at `shared/`.

---

## What happened

The iOS app has been significantly enhanced with new budgeting features. This repo documents **everything the web team needs to know** about the database, document shapes, computed values, and a new feature request.

---

## What you need to do

### 1. Add this repo as a submodule to your web repo

```bash
git submodule add https://github.com/shnx/rs-shared-contract.git shared
```

### 2. Read these files in order

| File | What it covers |
| --- | --- |
| `docs/DATA_CONTRACT.md` | Complete Firestore reference — every collection, every field, every computed formula |
| `docs/SHARED_LINK_GUIDE.md` | New feature: shareable read-only budget dashboard links (needs web implementation) |
| `docs/CATEGORY_TAXONOMY.md` | New two-tier category system (6 broad categories with sub-categories) |
| `schemas/*.json` | Machine-readable JSON Schemas for all document types |

### 3. Reply to GitHub issue #1

Go to: https://github.com/shnx/rs-shared-contract/issues/1

Reply with:
- ✅ Confirm you have read and understood the data contract
- Any questions about document shapes, computed values, or the collection prefix logic
- Whether the `sharedViews` collection + Express endpoint for shareable links is something you can implement, and an estimated timeline
- Any conflicts with existing web-side schemas (e.g. field names that differ)

### 4. Going forward

When you change any Firestore schema on the web side:
1. Update this repo (clone it separately, make changes, push)
2. Bump `CHANGELOG.md`
3. Both repos run `cd shared && git pull` to get the update

The iOS dev will do the same when iOS-side schema changes happen.

---

## Key points to be aware of

### Collection prefix
- iOS writes to `ady_*` prefixed collections (e.g. `ady_transactions`, `ady_periods`)
- Web writes to **un-prefixed** collections (e.g. `transactions`, `periods`)
- iOS reads **both** and merges by document ID, keeping the newer `updatedAt`
- Your queries should hit the **un-prefixed** collections

### ADY project resolution
The project ID is found by looking for a project whose `name` (or `title`) contains "ady" (case-insensitive) in the `projects` collection. If only one project exists, it's assumed to be ADY.

### Currency
All amounts are normalized to JOD for aggregation. USD → JOD conversion rate is **0.707**.

### New collection needed: `sharedViews`
For generating shareable read-only links to the budget dashboard. See `docs/SHARED_LINK_GUIDE.md` for the full spec.

Schema:
```jsonc
{
  "projectId":  "string",
  "createdBy":  "string",
  "token":      "string",        // Random UUID — used in the URL
  "label":      "string?",
  "expiresAt":  Timestamp?,
  "createdAt":  Timestamp,
  "revoked":    boolean
}
```

### Category format
Categories may now be stored as `"Broad > Sub"` (e.g. `"Hardware > Components"`). The web should handle this format in any category display or filtering.

### New computed values (iOS already implements these)
- **Cash flow timeline** — running balance per transaction within a period
- **Carry-over debt** — negative balances carry forward across periods
- **Spending forecast** — projected spend for planned periods based on historical data

All formulas are documented in `docs/DATA_CONTRACT.md`.

---

## TL;DR

1. Add this repo as a submodule
2. Read the docs
3. Reply to [issue #1](https://github.com/shnx/rs-shared-contract/issues/1)
4. Let us know about the shareable link feature timeline
