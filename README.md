# RS Shared Contract

This repository is the **single source of truth** for the data contract between the iOS app (`RS-IOS-APP`) and the web app. Both repos include it as a git submodule at `shared/`.

## What's here

| Path | Purpose |
| --- | --- |
| `docs/SHARED_LINK_GUIDE.md` | Full guide for the shareable budget dashboard link feature |
| `docs/DATA_CONTRACT.md` | Complete Firestore collection + document shape reference |
| `docs/CATEGORY_TAXONOMY.md` | Two-tier category system (broad + sub-categories) |
| `schemas/transaction.json` | JSON Schema for `transactions` / `ady_transactions` documents |
| `schemas/period.json` | JSON Schema for `periods` / `ady_periods` documents |
| `schemas/portal-document.json` | JSON Schema for `documents` / `ady_documents` |
| `schemas/period-summary.json` | JSON Schema for `periodSummaries` |
| `schemas/pdf-receipt-meta.json` | JSON Schema for `ady_pdf_meta` |
| `schemas/shared-view.json` | JSON Schema for `sharedViews` (new — for shareable links) |
| `schemas/category-taxonomy.json` | Machine-readable category taxonomy with keywords |
| `CHANGELOG.md` | Record breaking schema changes here |

## How to use

### In the iOS repo
```bash
git submodule add https://github.com/shnx/rs-shared-contract.git shared
```

### In the web repo
```bash
git submodule add https://github.com/shnx/rs-shared-contract.git shared
```

### Updating after changes
```bash
cd shared && git pull origin main
cd .. && git add shared && git commit -m "Update shared contract"
```

## Rules

1. **Never edit files in `shared/` directly from the parent repo** — always make changes in a clone of `rs-shared-contract`, push, then update the submodule.
2. **Bump `CHANGELOG.md`** for any schema change (added field, removed field, type change).
3. **Both repos must update the submodule** after a schema change — CI should verify.
