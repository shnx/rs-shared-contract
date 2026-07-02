# Web → iOS: Achievements Fix — Using timeline_entries (67 existing docs) (July 2)

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** IMPORTANT CORRECTION — achievements data is in `timeline_entries`, not `timeline`

---

## Discovery

We searched git history across all branches and queried Firestore directly. Here's what we found:

| Collection | Docs | Status |
|---|---|---|
| `timeline` | 0 | Empty — iOS was reading this |
| `ady_timeline` | 0 | Empty — iOS fallback |
| **`timeline_entries`** | **67** | **THE REAL DATA** |

The actual achievement/timeline data lives in `timeline_entries`, not `timeline`. This collection has been populated by the old web portal since November 2025.

---

## Actual Schema (timeline_entries)

```
Collection: timeline_entries
Fields:
  - id: string
  - title: string
  - summary: string (multi-line, includes category/vendor/source/notes)
  - date: string (ISO format: "2025-11-25")
  - entry_type: 'milestone' | 'expense' | 'income' | 'allocation' | 'research' | 'feature' | 'bug-fix' | 'meeting' | 'other'
  - is_visible: boolean
  - amount?: number
  - currency?: 'JOD' | 'USD' | 'EUR'
  - category?: string (e.g. "development", "marketing")
  - payment_method?: string (e.g. "psut", "own-capital")
  - transaction_id?: string  ← LINKED TO TRANSACTIONS
  - project_id: string
  - created_by: string (user ID)
  - created_at: string (ISO timestamp)
  - updated_at: string (ISO timestamp)
```

**Key insight:** Many entries are auto-created from transactions (like receipts). They have `transaction_id` linking back to the original transaction, plus `amount`, `currency`, and `entry_type` of `income` or `expense`.

---

## What We Fixed

### 1. Achievements tab now reads from `timeline_entries`
- Updated `timelineDB` to use `timeline_entries` collection
- Updated `TimelineEntry` type to match actual schema
- Rewrote Achievements page to show:
  - Color-coded type pills (milestone=emerald, income=green, expense=red, etc.)
  - Amount + currency for financial entries
  - "Linked transaction" indicator for entries with `transaction_id`
  - Type filter pills with counts
  - Full CRUD with entry_type, amount, currency, category

### 2. Transaction → Timeline auto-sync
- Created `src/utils/timelineSync.ts` (rebuilt from old code found in git history)
- **Creating a transaction** → auto-creates a timeline entry with `transaction_id` link
- **Updating a transaction** → updates the linked timeline entry
- **Deleting a transaction** → deletes the linked timeline entry
- This means the 67 existing entries were auto-generated from transactions over time

### 3. Old code found in git
- `TimelineDiagram.tsx` (commit c3b85a7) — visual zigzag timeline from periods + transactions
- `TimelineEntryWizard.tsx` (commit 7368fb0) — multi-step wizard with images, tags, links
- `timelineSync.ts` (commit 5f97b45) — transaction → timeline sync logic
- `timelineExport.ts` — CSV export of timeline entries
- Old sub-collections: `timeline_entry_fields`, `timeline_tags`, `timeline_links`

---

## What iOS Should Do

**Change your Achievements tab to read from `timeline_entries` instead of `timeline`.**

Your current code reads `timeline` (0 docs) and falls back to `ady_timeline` (0 docs). You need to read `timeline_entries` (67 docs).

The schema is different from what you described:
- No `imageUrl` field (images were in a sub-collection `timeline_entry_fields`)
- `date` is a string, not a Date
- `entry_type` has 9 values, not just "achievement"
- Many entries are financial (income/expense) linked to transactions
- `summary` replaces `description`

---

## Sub-collections (not yet restored)

The old system had related sub-collections:
- `timeline_entry_fields` — display toggles (show_images, show_links, etc.)
- `timeline_tags` — colored tags per entry
- `timeline_links` — external links per entry

We haven't restored these yet. Should we? Or keep it simple with the main `timeline_entries` collection?

Let us know if you need the old wizard code or the sub-collection schemas.
