# Achievements ↔ Transactions Linkage Spec

> **The problem:** Achievements (timeline entries) and transactions overlap — income/expense achievements ARE financial transactions with a story wrapper. This spec explains the link and how to present both without duplicating data.

---

## 1. The Link

### How They Connect

```
timeline_entries/{entryId}
  └── transaction_id: "abc123"  ← points to a transaction

ady_transactions/{transactionId}
  └── evidenceFiles: [...]       ← receipts/bills
  └── amount, currency, vendor, fundingSource, etc.
```

When a timeline entry has `transaction_id` set, it is **linked** to a transaction. The timeline entry provides the **narrative** (title, summary, image, entry_type), and the transaction provides the **financial data** (amount in JOD, evidence, vendor, funding source).

### Entry Types and Their Relationship to Transactions

| entry_type | Should link to transaction? | What it represents |
|---|---|---|
| `income` | **YES** — must link | Money received (grant installment, payment) |
| `expense` | **YES** — must link | Money spent (equipment, services) |
| `allocation` | **YES** — must link | Budget allocated to a category |
| `milestone` | No | Project milestone (prototype done, deployment) |
| `research` | No | Research activity |
| `feature` | No | New feature built |
| `bug-fix` | No | Bug fixed |
| `meeting` | No | Meeting (including with Sari) |
| other | No | Anything else |

### The Golden Rule

**Never duplicate financial data in the timeline entry.** If `transaction_id` is set:
- Amount comes from the **transaction** (`amountJOD`), not from the timeline entry's `amount` field
- Evidence/receipts come from the **transaction** (`evidenceFiles`), not from the timeline entry
- The timeline entry's `amount` field is a fallback for display ONLY when `transaction_id` is missing

---

## 2. How the iOS App Presents Them (After Fix)

### Achievements Tab
- Shows ALL timeline entries as a visual timeline (story view)
- For each entry, shows: date, type icon, title, summary, image
- **NEW:** If linked to a transaction:
  - Shows real amount in JOD (from transaction, with +/- sign and color)
  - Shows vendor and category from the transaction
  - Shows 📎 badge if the transaction has evidence
  - "View Receipt" button opens the evidence viewer
- **NEW:** If income/expense entry has NO transaction link:
  - Shows red "NOT IN BUDGET" badge
  - Detail view warns: "This entry is not linked to a transaction. Add it in the Transactions tab."
- **NEW:** Summary card at top shows counts: Linked / With Receipt / Unlinked

### Transactions Tab
- Shows ALL transactions as a financial management list
- For each transaction: date, type, amount, category, vendor, funding source, evidence
- This is where you ADD, EDIT, DELETE transactions
- This is where you ASSIGN transactions to periods
- This is where you UPLOAD receipts/evidence

### How They Complement Each Other (No Duplication)

| Feature | Achievements Tab | Transactions Tab |
|---|---|---|
| Purpose | Story / narrative | Financial management |
| Data source | `timeline_entries` | `ady_transactions` |
| Shows amounts | Yes (from linked transaction) | Yes (from transaction) |
| Shows evidence | Yes (from linked transaction) | Yes (from transaction) |
| Can add/edit/delete | No | Yes |
| Can assign to periods | No | Yes |
| Can upload receipts | No | Yes |
| Shows images | Yes (timeline image) | No |
| Shows entry type (milestone, etc.) | Yes | No |
| Shows vendor | Yes (from linked tx) | Yes |
| Shows funding source | No | Yes |
| Sort/group by period | No | Yes |
| Export to PDF/CSV | No | Yes |

**The key insight:** The Achievements tab is a **read-only story view** that pulls financial details from linked transactions. The Transactions tab is the **management tool** where financial data lives. No financial data is duplicated — it's always sourced from the transaction.

---

## 3. What the Web Developer Needs to Do

### Achievements Page (`/achievements` or ADY project achievements section)

1. **Load both collections:**
   - `timeline_entries` (where `is_visible != false`)
   - `ady_transactions` (all transactions for the project)

2. **For each timeline entry:**
   - If `transaction_id` is set, find the matching transaction
   - Display amount from the **transaction** (not from the entry's `amount` field)
   - Show evidence/receipt button if the linked transaction has `evidenceFiles`
   - Show vendor from the linked transaction

3. **For income/expense entries without `transaction_id`:**
   - Show red "Not in Budget" badge
   - Show the entry's own `amount` field as a fallback (with a note that it's not in the budget)

4. **Summary card:**
   - Count of linked entries
   - Count of entries with receipts
   - Count of unlinked financial entries (should be 0 ideally)

5. **Evidence viewer:**
   - When user clicks "View Receipt" on an achievement
   - Fetch the linked transaction from `ady_transactions/{transactionId}`
   - Decode `evidenceFiles[].base64Data` and display inline (PDF viewer or image)
   - Provide download button

### Transactions Page (already exists, just ensure alignment)

1. For transactions that have a linked timeline entry:
   - Show a small "📖 Story" badge or icon
   - Optionally link to the achievement page

2. When creating a transaction from the web:
   - Optionally create a timeline entry with `transaction_id` pointing to the new transaction
   - Set `entry_type` based on transaction type:
     - `payment` / `check` / `revenue` → `entry_type = "income"`
     - `bill` / `cost` → `entry_type = "expense"`

---

## 4. Firestore Schema Summary

### timeline_entries (Achievements)

```json
{
  "title": "PSUT Grant Installment Received",
  "summary": "Second installment of the PSUT Venture Lab grant",
  "date": "2026-06-15T00:00:00Z",
  "entry_type": "income",
  "is_visible": true,
  "category": "Grant",
  "project_id": "ady_project_id",
  "transaction_id": "abc123",          // ← THE LINK
  "amount": 2000,                       // fallback only, use transaction's amountJOD when linked
  "currency": "JOD",
  "image_url": "gs://bucket/timeline/...",
  "created_by": "Mohammad Shannak",
  "created_at": "2026-06-15T10:00:00Z"
}
```

### ady_transactions (Transactions)

```json
{
  "id": "abc123",
  "type": "payment",                    // income type
  "amount": 2000,
  "currency": "JOD",
  "date": "2026-06-15T00:00:00Z",
  "category": "Grant",
  "description": "PSUT Venture Lab - 2nd installment",
  "vendor": "PSUT Venture Lab",
  "funding_source": "psut",
  "evidence_files": [
    {
      "id": "file1",
      "file_name": "grant_receipt.pdf",
      "file_type": "application/pdf",
      "file_size": 123456,
      "base64_data": "JVBERi0xLjQ..."
    }
  ],
  "evidence_attached": true,
  "project_id": "ady_project_id",
  "period_id": "period_1"
}
```

---

## 5. Rules for Creating Linked Entries

### When adding a transaction in the Transactions tab:
1. Create the transaction in `ady_transactions`
2. Optionally create a timeline entry in `timeline_entries` with:
   - `transaction_id` = the new transaction's ID
   - `entry_type` = "income" (for payment/check/revenue) or "expense" (for bill/cost)
   - `amount` = same as transaction amount (fallback)
   - `currency` = same as transaction currency
   - `title` = a human-readable title (e.g., "Equipment purchased")
   - `summary` = optional description
   - `is_visible` = true

### When adding an achievement in the Achievements tab:
1. If it's an income/expense/allocation type, ALSO create a transaction
2. Set `transaction_id` on the timeline entry to point to the new transaction
3. The transaction holds the financial data; the timeline entry holds the story

### When deleting a transaction:
1. If a timeline entry links to it, either:
   - Delete the timeline entry too, OR
   - Set `transaction_id` to null on the timeline entry (it becomes unlinked)

---

## 6. Migration: Fixing Existing Unlinked Entries

If there are existing income/expense timeline entries without `transaction_id`:

1. For each unlinked financial entry:
   - Check if a transaction exists with the same date + amount + type
   - If found, set `transaction_id` on the timeline entry to point to it
   - If not found, create a transaction from the timeline entry's data, then link

2. The iOS app shows "NOT IN BUDGET" badge for these until they're linked
