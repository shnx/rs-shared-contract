# Firestore Data Contract

Complete reference for every collection and document shape shared between the iOS app (`RS-IOS-APP`) and the web app.

## Firebase Project

- **Project ID:** `robotics-website-5593f`
- Both apps read/write the same Firestore project.

## Collection Prefix Logic

iOS uses `CollectionConfig.prefix = "ady_"` for its own writes. The web writes to **un-prefixed** collections. iOS reads **both** and merges by document ID, keeping the most recently updated copy.

| Un-prefixed (web) | Prefixed (iOS) | Merge Strategy |
| --- | --- | --- |
| `transactions` | `ady_transactions` | Deduplicate by ID, keep newer `updatedAt` |
| `periods` | `ady_periods` | Deduplicate by ID, keep newer `updatedAt` |
| `documents` | `ady_documents` | Try un-prefixed first, fall back to prefixed |

## ADY Project Resolution

Read `projects` collection. Find the document whose `name` (or `title`) contains "ady" (case-insensitive). If only one project exists, assume it's ADY. Cache the ID.

---

## Collections

### `projects`

No strict schema — iOS reads raw dictionaries. Key fields:

```jsonc
{
  "name":    "string",   // or "title"
  "status":  "string?",
  "projectId": "string?"
}
```

### `transactions` / `ady_transactions`

```jsonc
{
  "projectId":       "string",            // REQUIRED — ADY project id
  "storeId":         "string?",           // Legacy
  "type":            "cost | revenue",    // Also: "payment", "bill", "check" (legacy)
  "category":        "string?",           // Free text, may be "Broad > Sub"
  "amount":          number,              // Decimal or Double
  "currency":        "JOD | USD",         // Defaults to JOD
  "date":            Timestamp,           // Also: string/epoch/map (iOS decodes all)
  "description":     "string?",           // Primary human-facing text
  "note":            "string?",
  "notes":           "string?",           // Alias for note
  "periodId":        "string?",           // Links to a Period
  "vendor":          "string?",
  "fundingSource":   "psut | own-capital | revenue",
  "evidenceAttached": boolean?,
  "evidenceFiles":   [EvidenceFile],      // See below
  "installmentId":   "string?",
  "subscriptionId":  "string?",
  "attachmentUrl":   "string?",
  "createdAt":       Timestamp?,
  "updatedAt":       Timestamp?
}
```

#### EvidenceFile

```jsonc
{
  "id":          "string",
  "fileName":    "string",
  "fileType":    "application/pdf | image/jpeg",
  "fileSize":    number?,
  "uploadDate":  Timestamp?,
  "base64Data":  "string"                 // "data:mime;base64,..." or raw base64
}
```

### `periods` / `ady_periods`

```jsonc
{
  "projectId":      "string",
  "storeId":        "string?",             // Legacy
  "name":           "July 2026",
  "startDate":      Timestamp,
  "endDate":        Timestamp,
  "status":         "planned | active | completed",
  "isClosed":       boolean?,             // Legacy mirror of completed
  "allocatedFunds": number?,
  "plannedTotal":   number?,
  "plannedItems":   [PlannedItem],        // See below
  "spentAmount":    number?,              // iOS recomputes on save
  "revenue":        number?,              // iOS recomputes on save
  "netPosition":    number?,
  "fundName":       "string?",
  "fundTotal":      number?,
  "notes":          "string?",
  "createdAt":      Timestamp?,
  "updatedAt":      Timestamp?
}
```

#### PlannedItem

```jsonc
{
  "id":       "string",
  "category": "Hardware",
  "amount":   number,
  "note":     "string?"
}
```

#### Status decoding (lenient)

| Stored value | Decoded as |
| --- | --- |
| `"planned"` | planned |
| `"completed"`, `"closed"`, `"done"` | completed |
| anything else | active |

### `documents` / `ady_documents`

```jsonc
{
  "projectId":            "string?",
  "fileName":             "string?",
  "fileType":             "application/pdf | image/jpeg",
  "fileSize":             number?,        // May be string or Int
  "category":             "string?",
  "base64Data":           "string?",
  "extractedText":        "string?",
  "linkedTransactionId":  "string?",      // THE link to a transaction
  "uploadedBy":           "string?",
  "uploadedAt":           Timestamp?
}
```

### `periodSummaries`

Written by iOS after each data load. Pre-computed snapshots.

```jsonc
{
  "projectId":        "string?",
  "periodId":         "string?",          // Non-nil for period summaries
  "monthKey":         "string?",          // Non-nil for monthly roll-ups ("yyyy-MM")
  "label":            "string",
  "totalExpense":     number,
  "totalIncome":      number,
  "netPosition":      number,
  "transactionCount": number,
  "byType":           [SummaryTypeEntry],
  "byCategory":       [SummaryCategoryEntry],
  "computedAt":       Timestamp
}
```

#### SummaryTypeEntry

```jsonc
{ "type": "cost", "total": number, "count": number }
```

#### SummaryCategoryEntry

```jsonc
{ "category": "Hardware", "planned": number, "actual": number }
```

### `ady_pdf_meta`

OCR-extracted receipt data, written by iOS on-device.

```jsonc
{
  "projectId":         "string?",
  "transactionId":     "string?",
  "fileName":          "string?",
  "amount":            number?,
  "currency":          "string?",
  "date":              Timestamp?,
  "vendor":            "string?",
  "invoiceNumber":     "string?",
  "suggestedCategory": "string?",
  "suggestedProject":  "string?",
  "confidence":        number?,
  "method":            "on-device",
  "extractedText":     "string?",
  "rawJSON":           "string?",
  "analyzedAt":        Timestamp?
}
```

### `sharedViews` (NEW — for shareable links)

```jsonc
{
  "projectId":  "string",
  "createdBy":  "string",
  "token":      "string",            // Random UUID — used in the URL
  "label":      "string?",
  "expiresAt":  Timestamp?,
  "createdAt":  Timestamp,
  "revoked":    boolean
}
```

---

## Computed Values

### Classification

- **Expense** = `type == "cost"` OR `type == "bill"`
- **Income** = `type == "revenue"` OR `type == "payment"` OR `type == "check"`

### Currency

- **JOD** = use as-is
- **USD → JOD** = multiply by `0.707`

### Project-level

| Metric | Formula |
| --- | --- |
| Total Spent | Σ `amountJOD` where `isExpense` |
| Total Income | Σ `amountJOD` where `isIncome` |
| PSUT Spent | Σ `amountJOD` where `type=="cost"` AND `fundingSource=="psut"` |
| PSUT Received | Σ `amountJOD` where `isIncome` AND `fundingSource=="psut"` |
| Fund Total | 10,000 JOD (hardcoded in iOS @AppStorage) |
| In Cash | `max(0, psutReceived - psutSpent)` |
| To Receive | `max(0, fundTotal - psutReceived)` |
| Is Overspent | `totalSpent > totalIncome` |
| Overspend Amount | `max(0, totalSpent - totalIncome)` |

### Period-level

| Metric | Formula |
| --- | --- |
| Actual Expense | Σ `amountJOD` where `isExpense` AND in period |
| Actual Income | Σ `amountJOD` where `isIncome` AND in period |
| Net Cash | `actualIncome - actualExpense` |
| Avg Daily Spend | `actualExpense / max(1, daysInPeriod)` |
| Avg Monthly Spend | `actualExpense / max(1, monthsInPeriod)` |
| Planned Total | Σ `plannedItems.amount` → `plannedTotal` → `allocatedFunds` → 0 |

**Transaction belongs to period if:**
- `transaction.date` is within `[period.startDate, period.endDate]` (inclusive of entire end day), OR
- `transaction.periodId == period.id`

### Cash flow timeline

For each transaction in period, sorted by `date` ascending:
1. `delta` = `+amountJOD` if income, `-amountJOD` if expense
2. `runningBalance` = cumulative sum of deltas
3. `isOverspend` = `runningBalance < 0`

### Carry-over debt

Sort all periods by `startDate` ascending. For each:
- `periodNet` = `actualIncome - actualExpense`
- `carryIn` = cumulative net from all prior periods
- `carryOut` = `carryIn + periodNet`
- `isDebt` = `carryOut < 0`

### Spending forecast (for planned periods)

1. Find up to 3 most recent **completed** periods
2. For each: `dailySpend = actualExpense / daysInPeriod`
3. Average those daily rates
4. `projectedSpend = avgDailySpend * targetPeriodDays`
5. `projectedIncome` = same method using income
6. Confidence = `min(1.0, historicalCount / 3.0 * 0.7 + 0.3)`
7. If no completed periods: fall back to overall avg daily spend, 30% confidence
