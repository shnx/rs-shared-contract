# Shareable Budget Dashboard — Web Integration Guide

## Overview

The iOS app manages a full budgeting system for the **ADY project**. This guide describes how to build a **shareable link** that opens a read-only web page showing the same financial tables the iOS app displays.

## Architecture

### New Firestore collection: `sharedViews`

```jsonc
{
  "projectId":  "string",
  "createdBy":  "string",
  "token":      "string",        // Random UUID — the shareable token in the URL
  "label":      "string?",
  "expiresAt":  Timestamp?,
  "createdAt":  Timestamp,
  "revoked":    boolean,
  "revokedAt":  Timestamp?,      // Set when link is deactivated

  // ── Fund (PSUT grant) — matches the iOS fund card exactly ──
  "fund": {
    "name":        "string",     // e.g. "PSUT Venture Lab"
    "total":       number,       // JOD — total grant amount (e.g. 10000)
    "received":    number,       // JOD — PSUT-tagged income received so far
    "spent":       number,       // JOD — PSUT-tagged expenses (cost + bill)
    "inCash":      number,       // JOD — max(0, received - spent) — money in hand
    "toReceive":   number,       // JOD — max(0, total - received) — still expected
    "receivedPct": number        // 0–100 — percentage of grant received
  },

  // ── Embedded financial summary (computed on iOS, matches overview) ──
  "summary": {
    "totalIncome":      number,   // JOD — all income (payment + check + revenue types)
    "totalSpent":       number,   // JOD — all expenses (bill + cost types)
    "netBalance":       number,   // totalIncome - totalSpent (can be negative = debt)
    "transactionCount": number,
    "periodCount":      number,
    "isOverspent":      boolean,  // true if totalSpent > totalIncome
    "overspendAmount":  number,   // JOD — how much over (0 if not overspent)
    "generatedAt":      number    // epoch ms — when the summary was computed
  },
  "periods": [
    {
      "id":               "string",
      "name":             "string",
      "startDate":        number,  // epoch ms
      "endDate":          number,  // epoch ms
      "allocatedFunds":   number,  // JOD
      "income":           number,  // JOD — actual income in this period
      "spent":            number,  // JOD — actual expenses in this period
      "net":              number,  // income - spent
      "transactionCount": number
    }
  ],

  // ── Carry-over timeline (chronological cumulative balance across periods) ──
  "carryOverTimeline": [
    {
      "periodId":    "string",
      "periodName":  "string",
      "periodNet":   number,   // JOD — this period's income - expense
      "carryIn":     number,   // JOD — cumulative balance before this period
      "carryOut":    number,   // JOD — cumulative balance after this period (negative = debt)
      "isDebt":      boolean   // true if carryOut < 0
    }
  ],

  // ── Monthly buckets (grouped by yyyy-MM) ──
  "monthlyBuckets": [
    {
      "id":      "string",    // yyyy-MM
      "title":   "string",    // e.g. "June 2026"
      "income":  number,      // JOD
      "expense": number,      // JOD
      "net":     number,      // JOD — income - expense
      "count":   number       // transaction count
    }
  ],

  // ── Transactions with evidence (for receipt viewing on web) ──
  "transactions": [
    {
      "id":            "string",
      "type":          "string",    // cost, bill, revenue, payment, check
      "isExpense":     boolean,
      "isIncome":      boolean,
      "description":   "string",
      "vendor":        "string",
      "amount":        number,      // JOD (already converted from USD if needed)
      "currency":      "string",    // "JOD" or "USD"
      "date":          number,      // epoch ms
      "category":      "string",
      "fundingSource": "string",    // psut, own-capital, revenue
      "periodId":      "string",
      "note":          "string",
      "evidence": [                 // Receipt files — base64 embedded
        {
          "id":          "string",
          "fileName":    "string",
          "fileType":    "string",    // application/pdf, image/jpeg, etc.
          "base64Data":  "string"     // raw base64 — render as data URI
        }
      ]
    }
  ]
}
```

> **Important:** The web app should prefer the embedded `fund`, `summary`, `periods`,
> `carryOverTimeline`, `monthlyBuckets`, and `transactions` fields over recalculating
> from raw transactions. The iOS app classifies transactions differently than the web app:
>
> - **iOS income**: `payment` + `check` + `revenue` types
> - **iOS expense**: `bill` + `cost` types
> - **Web (old)**: only `revenue` as income, everything else as cost
>
> This mismatch caused the web dashboard to show wrong numbers (e.g. showing
> positive cash when actually in debt). Using the embedded fields ensures
> the shared dashboard matches the iOS app exactly.
>
> **Receipts/evidence:** The `transactions` array includes an `evidence` field per
> transaction. If the array is empty, there is no receipt. If it has entries, each
> contains `base64Data` that can be rendered as a data URI (`data:{fileType};base64,{base64Data}`).
> Do NOT show any icon or indicator for whether a receipt exists — only show it when
> the user opens transaction details.

### URL format

```
https://yourdomain.com/shared/{token}
```

### Data fetching (client-side)

**Updated:** The web app now fetches data **directly from Firestore** on the client side (no Express endpoint needed for Firebase Hosting).

**Client-side logic:**
1. Query `sharedViews` where `token == :token` AND `revoked != true`
2. If `expiresAt` exists and is in the past → show expired message
3. Fetch `projectId` from the document
4. Query `transactions` + `ady_transactions` where `projectId == <id>` (merge by ID, keep newer `updatedAt`)
5. Query `periods` + `ady_periods` where `projectId == <id>` (merge by ID)
6. Query `documents` where `projectId == <id>`
7. Compute summaries client-side (same formulas as iOS)

### Response shape

```jsonc
{
  "project": { "id": "...", "name": "..." },
  "fund": {
    "total": 10000,
    "spent": 5400,
    "received": 6000,
    "inCash": 600,
    "toReceive": 4000
  },
  "totals": {
    "totalSpent": 5400,
    "totalIncome": 6000,
    "isOverspent": false,
    "overspendAmount": 0,
    "transactionCount": 42,
    "periodCount": 5
  },
  "periods": [
    {
      "id": "...",
      "name": "July 2026",
      "status": "active",
      "startDate": "...",
      "endDate": "...",
      "plannedTotal": 2000,
      "actualExpense": 1850,
      "actualIncome": 2000,
      "netCash": 150,
      "transactionCount": 12,
      "carryIn": 0,
      "carryOut": 150
    }
  ],
  "transactions": [ /* all transactions */ ],
  "carryOverTimeline": [ /* chronological carry-over entries */ ],
  "monthlyBuckets": [ /* yyyy-MM grouped summaries */ ],
  "categoryBreakdown": [ /* consolidated by broad category */ ]
}
```

## Web Page Sections (matching iOS)

### Project overview page

1. **Fund Card** — PSUT grant progress bar (spent / in cash / to receive)
2. **Quick Stats** — Total Spent, Total Income, Transactions count, Periods count
3. **Overspend Alert** (if `totalSpent > totalIncome`)
4. **Carry-over Timeline** — chronological balance across all periods
5. **Recent Transactions** — last 5
6. **Spending Insights** — by funding source + top categories (consolidated)
7. **Periods List** — each period with name, date range, status, planned vs actual
8. **Monthly Breakdown** — grouped by `yyyy-MM`

### Period detail page (optional drill-in)

1. **Header** — name, status, date range
2. **Period Stats** — planned, actual, variance, income
3. **Cash Flow Card** — running balance timeline, avg/day, avg/month, overspend warnings
4. **Carry-over Card** — carried in, this period net, carried out
5. **Forecast Card** (if status == "planned") — projected spend/income with confidence
6. **Spending by Category** — consolidated breakdown with bars
7. **Planned vs Actual** — per-category table
8. **By Funding Source** — breakdown
9. **Transactions List** — all transactions in the period

## Security

- `sharedViews` collection: **public read** — the web client reads it directly (token is a random UUID, doc only contains `projectId` + `token` + `label` + `expiresAt`)
- `transactions`, `periods`, `documents` collections: still require auth — the shared dashboard only reads `sharedViews` client-side; financial data is fetched via the same public read rules as `venture_reports` (if needed, we can switch to a Cloud Function proxy)
- Page is **read-only** — no edit/delete/create
- Evidence files (base64) can be large — exclude from payload or lazy-load per transaction

## iOS-side changes

Add a "Share" button in `ProjectWorkspaceView` toolbar that:
1. Creates a `sharedViews` document in Firestore with a random token
2. Presents the shareable URL (copy to clipboard / share sheet)

## Firestore Security Rules for `sharedViews`

```
match /sharedViews/{docId} {
  // Public read — web client fetches directly for Firebase Hosting
  allow read: if true;
  allow create: if request.auth != null;
  allow update, delete: if request.auth != null;
}
```
