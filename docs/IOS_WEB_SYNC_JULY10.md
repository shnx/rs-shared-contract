# iOS → Web Team Sync — July 10

## Summary of iOS Changes (July 9–10)

This document covers all iOS-side changes pushed since the last sync. The web team should review and align where needed.

---

## 1. Project Consolidation (Aligned with Web Team)

The iOS app now matches the web team's 3-project consolidation exactly:

| Project | Key | Tabs |
|---|---|---|
| ADY | `ady` | Dashboard, Budget, Transactions, Payments, **Insights** (new), Timeline, Documents |
| Website Tools | `websiteTools` | Submissions, My Track, CRM, LaTeX, Carousel, Careers, Messages |
| Riad Shannak | `riadShannak` | Buildings, Documents |

- **Removed from iOS:** `studentTrack` project (merged into Website Tools as "My Track" tab)
- **ADY removed from main tab bar** — now accessed only through the Projects hub
- Projects hub shows 3 cards (ADY, Website Tools, Riad Shannak) instead of loading from Firestore `projects` collection (which was showing untitled projects)

---

## 2. Main Tab Bar Cleanup

Removed from the admin main tab bar:
- **Submissions** — available under Website Tools project only
- **Achievements** — removed entirely (no value)
- **Report Generator** — removed (broken, not generating proper reports)

Admin main tabs are now: **Projects, Screenshot, LaTeX, Carousel, Roles**

---

## 3. New ADY "Insights" Tab

A new tab has been added to the ADY project. This is iOS-only for now — the web team may want to build an equivalent.

### What it does

**Category Breakdown:**
- Groups all expense transactions using `CategoryEngine.consolidatedBreakdown`
- Shows 6 broad categories (Hardware, Software, Personnel, Operations, Fees, Funding)
- Each category shows: icon, transaction count, percentage, total amount, progress bar

**Recurring Payments:**
- Lists all transactions where `isRecurring == true`
- Shows: label, recipient name, frequency, category, amount, date
- Can remove recurring flag with a tap

**Add Recurring Payment:**
- Form to create a new recurring transaction in Firestore
- Fields: label, amount (JOD), frequency (weekly/biweekly/monthly/quarterly/yearly), start date
- Employee picker — loads from `systemUsers` where `role == "employee"`, auto-fills recipient name
- Category picker using `CategoryEngine.taxonomyNames`
- Saves to `ady_transactions` (via `WebTransactionsRepository`)

**AI Detect Recurring:**
- Sends transaction summaries to Gemini 2.5 Flash
- AI identifies transactions that appear to be recurring (same vendor/description at regular intervals, similar amounts, salary/subscription keywords)
- Marks detected transactions as `isRecurring = true` in Firestore
- Uses existing `Transaction` fields — no schema changes needed

### Firestore impact
- **No new collections** — uses existing `ady_transactions` / `transactions`
- **Fields used:** `isRecurring`, `recurrenceFrequency`, `recurrenceEndDate`, `recurrenceParentId`, `label`, `recipientName` — all already in the Transaction schema
- **New transactions** created by this feature have `source: "ios"`

### Web team action items
- Consider building a similar Insights view on the web portal
- The recurring payment fields (`isRecurring`, `recurrenceFrequency`) are already in the data contract — no schema changes needed
- If the web team builds recurring payment management, use the same fields for compatibility

---

## 4. Gemini Model Update

**All Gemini API calls updated from `gemini-2.0-flash` to `gemini-2.5-flash`.**

Gemini 2.0 Flash was shut down by Google on June 1, 2026. This was causing AI features to fail with model errors.

Affected files:
- `ScreenshotView.swift` — AI field detection for screenshots
- `CarouselCreatorView.swift` — AI carousel generation
- `MilestonesView.swift` — AI milestone generation
- `ADYInsightsTab.swift` — AI recurring detection (new)

**Web team:** If you're using Gemini on the web side, make sure to update to `gemini-2.5-flash` as well.

---

## 5. Screenshot Tool Updates

The Screenshot tool (admin-only, not project-scoped) has been rewritten:
- User picks their own base image from photo library via `PhotosPicker`
- Draggable overlay placeholders with live drag feedback
- Per-field controls: font weight, size, color, alignment, white stripe toggle
- AI field detection using Gemini Vision to auto-identify sensitive fields
- Preset save/load with full overlay data
- Export at original image resolution

This is iOS-only and has no web impact.

---

## 6. Transaction Fields Reference (for web team)

The iOS app now uses these Transaction fields for the Insights feature:

```
isRecurring: Bool?              // true for recurring payments
recurrenceFrequency: String?    // "weekly", "biweekly", "monthly", "quarterly", "yearly"
recurrenceEndDate: Date?        // when the recurring payment stops
recurrenceParentId: String?     // links recurring instances to original
label: String?                  // user-defined label: "Salary", "AI Development"
recipientName: String?          // person or entity receiving payment
source: String?                 // "ios" for iOS-created transactions
```

All of these are already in the shared data contract. No new fields were added.

---

## Action Items for Web Team

1. **Update Gemini model** to `gemini-2.5-flash` if using Gemini on web
2. **Consider building an Insights view** on the web portal (category breakdown + recurring payments)
3. **Confirm recurring payment fields** are handled in any web transaction forms
4. **No schema changes needed** — everything uses existing fields

---

## Git Commits (devin-1 branch)

| Commit | Description |
|---|---|
| `08fc468` | Lenient decoder for AIDetectedField |
| `6ba25f9` | Add ADY Insights tab |
| `7ac5b68` | Remove submissions/achievements/reports from main tabs; fix Gemini model |
| `6e03cc7` | Remove ADY from main tab bar; rewrite ScreenshotView |
| `f8bbf6f` | Fix .projects tab to use ProjectsHubView |
| `bda571f` | ScreenshotView PhotosPicker; 3-project structure |
