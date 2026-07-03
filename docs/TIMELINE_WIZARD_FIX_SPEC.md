# Spec: Timeline/Achievements Rename + Wizard Data Fix (July 3)

**From:** iOS team
**To:** Web team
**Re:** Three fixes — (1) rename "Achievements" to "Timeline" and show milestones only in PDF, (2) save wizard answers to sharedViews doc, (3) no transaction duplication in PDF.

---

## Issue 1: "Achievements" → "Timeline" (Milestones Only)

### Problem

The PDF report and shareable link showed ALL timeline entries (income, expense, allocation, milestone, other) under "Achievements & Work Completed". This is wrong:
- Expenses are not achievements
- Income entries are not achievements
- Only **milestones** should appear in the report timeline section

### What Changed (iOS)

**PDF Report** (`ReportPDFView.swift`):
- Section 03 renamed from "Achievements & Work Completed" → **"Timeline — Milestones"**
- Now filters `achievements` to only entries where `entryType == "milestone"`
- Each milestone shows: 🏆 title, date, description/summary, and attachment indicator if `imageUrl` is present
- Non-milestone entries (income, expense, allocation) are NOT shown in this section

**Shareable Link** (`ADYProjectView.swift` → `buildShareDocData`):
- `achievements` field renamed to `timeline` (full timeline, all entry types)
- New `milestones` field added (milestones only, `entryType == "milestone"`)
- Both fields include: `id`, `title`, `summary`, `date`, `entryType`, `imageUrl`, `category`

### What Web Team Should Do

1. **Rename "Achievements" tab/section to "Timeline"** in the web UI
2. **In the shared dashboard**, use the `milestones` field for the timeline section (not `achievements`)
3. **Keep `timeline` field** for the full timeline view (all entry types) if you show it elsewhere
4. **Backward compatibility**: old sharedViews docs may still have `achievements` field — fall back to it if `milestones` is missing

```javascript
// In shared dashboard component:
const milestones = data.milestones || data.achievements?.filter(e => e.entryType === 'milestone') || [];
const fullTimeline = data.timeline || data.achievements || [];
```

### Timeline Entry Types (Reference)

| `entryType` | Icon | Description |
|---|---|---|
| `milestone` | 🏆 | Project milestones — shown in PDF report |
| `income` | 💰 | Income events — NOT shown in PDF timeline |
| `expense` | 💸 | Expense events — NOT shown in PDF timeline |
| `allocation` | 🔄 | Fund allocations — NOT shown in PDF timeline |
| `meeting` | 📇 | Meetings — shown in Section 02 separately |
| `note` | 📝 | General notes — NOT shown in PDF timeline |
| `other` | • | Other — NOT shown in PDF timeline |

---

## Issue 2: Wizard Answers Not Saved to Shareable Link

### Problem

When you fill out the Venture Lab Report Wizard (challenges, actions taken, next steps, support needed, budget questions, overall summary), the data was:
- ✅ Used to generate the local PDF on device
- ❌ NOT saved to Firestore
- ❌ NOT included in the shareable link

So the shareable link showed financial data only — none of your wizard answers.

### What Changed (iOS)

**`buildShareDocData`** now accepts `wizardData` parameter and saves all wizard answers:

```javascript
// New field in sharedViews doc:
"wizardAnswers": {
  "challenges": "string",
  "actionsTaken": "string",
  "nextSteps": "string",
  "needsSupport": true/false/null,
  "supportDetails": "string",
  "unplannedExpensesExpected": true/false/null,
  "unplannedExpenseDetails": "string",
  "budgetSufficient": true/false/null,
  "budgetConcernDetails": "string",
  "overallSummary": "string",
  "reportScope": "All Periods" | "Specific Period" | "Custom Date Range",
  "selectedPeriodId": "string|null",
  "newPeriodName": "string",
  "predictedTransactions": [
    {
      "type": "cost|bill|payment|...",
      "amount": "string",
      "currency": "JOD|USD",
      "category": "string",
      "description": "string",
      "vendor": "string",
      "expectedDate": 1234567890000,  // epoch ms
      "fundingSource": "psut"
    }
  ]
}
```

**Flow fixed:**
1. Wizard completes → `onComplete(wizardData)` called
2. `ensureShareLink(wizardData:)` called → creates or updates sharedViews doc with wizard answers
3. Shareable link now includes all wizard answers

### What Web Team Should Do

1. **Read `wizardAnswers` from sharedViews doc** in the shared dashboard
2. **Display wizard answers** in the shared dashboard:
   - Challenges & Risks section
   - Support & Assistance section
   - Budget questions section
   - Overall Summary section
3. **Handle null values** — `needsSupport`, `unplannedExpensesExpected`, `budgetSufficient` can be `true`, `false`, or `null` (not answered)
4. **Show predicted transactions** if present — these are planned future spending items

---

## Issue 3: No Transaction Duplication in PDF

### How the PDF is now structured

| Section | Content | Source |
|---|---|---|
| 01 | Project Information | Static + period data |
| 02 | Meeting with Sari Awwad | `meetingNotes` |
| 03 | **Timeline — Milestones** | `timelineEntries` filtered to `entryType == "milestone"` only |
| 04 | Challenges & Risks | Wizard answers |
| 05 | Support & Assistance | Wizard answers |
| 06 | Budget & Spending | Budget summary table + **ALL transactions** in spending table + planned future spending |
| 07 | Budget Overview & Summary | Wizard answers |

**Section 03** shows milestones only (title, description, image indicator).
**Section 06** shows the full transaction log with receipt indicators (📎 or —).

**No duplication**: milestones are project events, transactions are financial records. They're different things shown in different sections.

### Transaction Table (Section 06)

Already existed and works correctly:
- Shows ALL transactions (income + expense), sorted by date
- Columns: Date, Type, Amount (JD), Category, Reason/Description, Receipt
- Receipt column shows 📎 if evidence exists, — if not
- Each transaction is a row in the table

### What Web Team Should Do

In the shared dashboard:
1. **Show milestones section** (from `milestones` field) — separate from transactions
2. **Show transactions table** (from `transactions` field) — with receipt/evidence indicators
3. **Make transactions clickable** — if `hasEvidence` is true, link to evidence viewer
4. **Make milestones clickable** — if `imageUrl` exists, show the image

---

## Summary of sharedViews Doc Changes

| Field | Before | After |
|---|---|---|
| `achievements` | All timeline entries | **Renamed to `timeline`** (all entries) |
| `milestones` | Did not exist | **NEW** — milestones only (`entryType == "milestone"`) |
| `wizardAnswers` | Did not exist | **NEW** — all wizard template answers |
| `timeline` | Did not exist | **NEW** — full timeline (replaces `achievements`) |

**Backward compatibility**: old docs still have `achievements`. Web should fall back:
```javascript
const milestones = data.milestones || filterMilestones(data.achievements) || [];
const fullTimeline = data.timeline || data.achievements || [];
const wizardAnswers = data.wizardAnswers || null;
```

---

*Reply by adding a new `.md` file to `shared/docs/`.*
