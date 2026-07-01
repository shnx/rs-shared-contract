# Venture Report & Achievements — Web Developer Handoff

> **Date:** 1 Jul 2026
> **From:** iOS app (RS-IOS-APP)
> **To:** Website developer (the-rs.com)

## Overview

The iOS app now generates **PSUT Venture Lab project reports** and saves them to Firestore. Each report gets a shareable link. The website needs to serve a public page that renders the report visually.

The iOS app also reads from the existing **`timeline`** Firestore collection for the new Achievements tab. No changes needed on the web side for that — the app just reads what's already there.

---

## New Firestore Collection: `venture_reports`

The iOS app writes report documents here when the user taps **Save & Get Link**.

### Document shape

```jsonc
{
  "projectId":          "string?",       // ADY project Firestore ID
  "reportTitle":        "string",        // e.g. "ADY Report — 1 Jul 2026"
  "projectTitle":       "string",        // e.g. "ADY — Autonomous Robotic Videography System"
  "projectManager":     "string",        // e.g. "Mohammad Shannak"
  "reportDate":         "Timestamp",     // when the report was filed
  "reportingPeriod":    "string",        // e.g. "1 Jun 2026 – 30 Jun 2026"
  "currentStage":       "string",        // e.g. "Development & Testing"

  // Meeting section
  "meetingDate":        "string",        // free-text date
  "meetingPoints":      "string",        // multi-line
  "meetingDecisions":   "string",        // multi-line

  // Progress section
  "achievements":       "string",        // multi-line
  "challenges":         "string",        // multi-line
  "challengeActions":   "string",        // multi-line
  "nextSteps":          "string",        // multi-line

  // Support section
  "needsIntervention":  "boolean",
  "interventionDetails":"string",        // present only if needsIntervention = true

  // Budget section
  "totalIn":            "number",        // JOD, decimal
  "totalOut":           "number",        // JOD, decimal
  "balance":            "number",        // JOD, decimal (totalIn - totalOut)
  "spendingLog":        "SpendingEntry[]",
  "plannedSpending":    "SpendingEntry[]",
  "unplannedExpected":  "boolean",
  "unplannedDetails":   "string",
  "budgetSufficient":   "boolean",
  "budgetConcernDetails":"string",
  "budgetSummary":      "string",

  // Generated content
  "latex":              "string",        // full LaTeX source
  "shareToken":         "string",        // 8-char unique token for the public URL

  "createdAt":          "Timestamp",
  "updatedAt":          "Timestamp"
}
```

### `SpendingEntry` shape

```jsonc
{
  "id":       "string",   // UUID
  "date":     "string",   // formatted date string
  "amount":   "string",   // formatted JOD amount
  "category": "string",
  "reason":   "string"
}
```

---

## Shareable Link

When a report is saved, the iOS app generates a link in this format:

```
https://the-rs.com/r/{shareToken}
```

The `shareToken` is an 8-character random string stored on the report document.

### What the website needs to do

1. **Add a route** for `/r/:token`
2. **Query Firestore**: `venture_reports` collection, `whereField("shareToken", isEqualTo: token)`
3. **Render the report** as a visual web page — not the raw LaTeX. Suggested layout:
   - Header with project title, manager, date, period
   - Meeting section (date, points, decisions)
   - Achievements section
   - Challenges & next steps section
   - Budget overview cards (total in, total out, balance)
   - Spending log table
   - Planned spending table
   - Budget summary
4. **Pull achievement photos** from the `timeline` Firestore collection and display them in a gallery or timeline below the report. Each timeline entry has:
   ```jsonc
   {
     "title":       "string",
     "description": "string?",
     "date":        "Timestamp?",
     "imageUrl":    "string?",   // URL or Firebase Storage path
     "category":    "string?"
   }
   ```
5. **Show receipts** — if you want to display transaction evidence, query `transactions` (or `ady_transactions`) for the same `projectId` and look at the `evidenceFiles` array (base64-encoded PDFs/images) and linked `documents`.

---

## Timeline Collection (Achievements)

The iOS app's Achievements tab reads from the **`timeline`** collection (un-prefixed, same one the website already uses). It also falls back to `ady_timeline` if the main collection is empty.

No new writes are made from the iOS app to `timeline` — it's read-only. The website can continue managing timeline entries as before.

---

## Summary of what's needed from the web developer

| Task | Priority |
| --- | --- |
| Add `/r/:token` route that reads `venture_reports` by `shareToken` | **High** |
| Build a visual report page (not LaTeX, but a styled HTML page) | **High** |
| Include achievement photos from `timeline` collection on the report page | **Medium** |
| Optionally show transaction receipts/evidence | **Low** |

No changes to existing collections or Firebase security rules are needed beyond allowing public read access to `venture_reports` by `shareToken`.
