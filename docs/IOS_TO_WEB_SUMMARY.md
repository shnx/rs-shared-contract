# iOS → Web Dev Team: Latest Findings & Shareable Link

## What We Built

### 1. Shareable Report Link (from iOS Reports tab)

The iOS app now has a **"Shareable Report Link"** section at the top of the ADY Reports tab. When the user taps "Create Link", the app:

1. Generates a UUID token
2. Writes a document to `sharedViews/{token}` in Firestore containing **all** the data the web needs to render a full report page
3. Shows the URL: `https://robotics-website-5593f.web.app/shared/{token}`

The user can also **Manage** the link (refresh data or revoke it).

### What's in the `sharedViews` document

Everything the web needs — no additional Firestore queries required to render the page:

```
sharedViews/{token}
├── projectId, token, label, createdAt, revoked
├── reportTemplate: "venture_lab"
├── fund: { name, total, received, spent, inCash, toReceive, receivedPct }
├── summary: { totalIncome, totalSpent, netBalance, transactionCount, periodCount, isOverspent, overspendAmount }
├── periods: [{ id, name, startDate, endDate, allocatedFunds, income, spent, net, transactionCount }]
├── carryOverTimeline: [{ periodId, periodName, periodNet, carryIn, carryOut, isDebt }]
├── monthlyBuckets: [{ id, title, income, expense, net, count }]
├── transactions: [{ id, date, type, category, description, amount, amountJOD, currency, vendor, fundingSource, note, isExpense, isIncome, hasEvidence, evidenceFiles, evidenceCount }]
├── meetings: [{ id, meetingDate, mainPoints, decisions }]
└── achievements: [{ id, title, summary, date, entryType, imageUrl }]
```

### What the web page should look like

A full Venture Lab report page (same 7-section template as the PDF) with:
- Project info, meetings with Sari, achievements, budget summary
- **Transaction log** showing ALL transactions (income + expense) with signed amounts
- **Clickable receipt buttons** — each transaction with `hasEvidence: true` shows a 📎 button that opens a modal with the receipt (fetch from `ady_transactions/{id}` and decode `evidenceFiles[].base64Data`)
- **Download as PDF** button at the top (use `window.print()` or `html2pdf.js`)
- PSUT grant card with progress bar (negative `inCash` shows red)
- Period summary table with carry-over

Full spec: `SHARED_REPORT_PAGE_SPEC.md`

---

### 2. Achievements ↔ Transactions Linkage

**Finding:** Achievements (timeline entries) and transactions are connected via a `transaction_id` field on the timeline entry. Income/expense achievements are essentially "story wrappers" around financial transactions.

**The link:**
```
timeline_entries/{entryId}
  └── transaction_id → ady_transactions/{transactionId}
```

**What we did in iOS:**
- Achievements tab now loads transactions alongside timeline entries
- If a timeline entry has `transaction_id`, the amount/vendor/evidence are pulled from the **linked transaction** (not duplicated from the entry's own `amount` field)
- Income/expense entries WITHOUT a transaction link show a red "NOT IN BUDGET" badge
- "View Receipt" button on achievements that have linked transactions with evidence
- Summary card shows: Linked count / With Receipt count / Unlinked count

**What this means for the web:**
- The achievements page should cross-reference `timeline_entries` with `ady_transactions`
- For linked entries, display amount from the transaction (not from the entry)
- Show receipt/evidence access on achievements when the linked transaction has evidence
- Flag unlinked financial entries (income/expense without `transaction_id`) — these are missing from the budget

Full spec: `ACHIEVEMENTS_TRANSACTION_LINKAGE.md`

---

### 3. Budget Calculations — You're Getting Wrong Numbers

We wrote a complete spec with every formula. The most common mistakes:

1. **`payment` type is INCOME, not expense** — only `bill` and `cost` are expenses
2. **All amounts must be converted to JOD** — `amountJOD = amount * 0.707` for USD
3. **`inCash` can be NEGATIVE** — don't clamp to zero
4. **Transactions are assigned to periods by DATE RANGE first**, not by `periodId`
5. **Carry-over is cumulative** — each period's net flows into the next
6. **`totalSpent` ≠ `psutSpent`** — totalSpent = all expenses; psutSpent = only `fundingSource == "psut"` expenses

Full spec: `ADY_CALCULATIONS_SPEC.md`

---

## Our Stance on Shareable Links

We want to keep building our own shareable links from the iOS side. Here's why and what we need from you:

### What we want to control from iOS:
- **Creating** the share link (generating token, writing `sharedViews` doc)
- **Refreshing** the data in the link (re-writing with latest numbers)
- **Revoking** the link (setting `revoked: true`)

### What we need from the web:
- **Render** the page at `/shared/{token}` using the data in the `sharedViews` document
- **Check `revoked`** on load — show "link expired" if true
- **Clickable receipts** — fetch evidence from `ady_transactions` when user clicks 📎
- **Download as PDF** button
- **Copy link** button

### We do NOT want the web to:
- Create shareable links on our behalf
- Modify the `sharedViews` document
- Delete or revoke links

### Future plans:
We may add more shareable views (dashboard snapshot, period-specific report, achievements-only page). For now, the single `sharedViews` doc with `reportTemplate: "venture_lab"` is all we need. When we add more templates, we'll set `reportTemplate` to a different value and update the spec.

---

## All Shared Docs

| Document | Purpose |
|---|---|
| `ADY_WEB_PORTAL_GUIDE.md` | Full overview of ADY data models, Firestore collections, iOS views |
| `ADY_CALCULATIONS_SPEC.md` | Every formula, every calculation, CSV formats, common mistakes |
| `SHARED_REPORT_PAGE_SPEC.md` | Layout and implementation spec for `/shared/{token}` page |
| `ACHIEVEMENTS_TRANSACTION_LINKAGE.md` | How achievements link to transactions, presentation rules |
| `SHARED_LINK_GUIDE.md` | (Existing) shared link management guide |

All docs are in the `shared/docs/` folder in the `rs-shared-contract` repo.
