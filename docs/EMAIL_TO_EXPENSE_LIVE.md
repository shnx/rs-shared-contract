# Email-to-Expense System — Live & Ready

**From:** Web team
**To:** iOS team
**Date:** July 3, 2026
**Re:** Email-to-expense system is deployed. Here's what we built and how to use it.

---

## What We Built

### 1. Cloud Function: `processInboundEmail`

A webhook endpoint that receives inbound emails from Resend when someone sends to `info@contact.the-rs.com`.

**URL:**
```
https://us-central1-robotics-website-5593f.cloudfunctions.net/processInboundEmail
```

**What it does:**
1. Receives the webhook payload from Resend (sender, subject, body, attachments)
2. Downloads PDF attachments and extracts text using `pdf-parse`
3. Runs regex extraction on (email body + PDF text) to find: amount, currency, date, vendor, category, description
4. Creates an `email_inbox` document in Firestore
5. Creates a draft transaction in `ady_transactions` with `reviewStatus: "pending_review"`, `source: "email"`
6. Deduplicates by `messageId` so the same email won't create duplicates

### 2. Web UI: Email Inbox Tab

A new admin-only tab in the portal that shows all received emails:
- Stats: total, pending, approved, rejected
- Filter by status
- Click any email → detail modal with:
  - Email body text
  - PDF receipt preview (embedded viewer)
  - Editable extracted data (amount, currency, description, vendor, category, date, period assignment)
  - Approve & Create / Reject / Ignore buttons
- Approving updates the transaction with the edited data and sets `reviewStatus: "approved"`

---

## Firestore Collections

### `email_inbox` (NEW)

```javascript
{
  id: "auto-generated",
  messageId: "resend-message-id",      // for dedup
  from: "store@arduino.com",
  fromName: "Arduino Store",
  to: "info@contact.the-rs.com",
  subject: "Order #12345 — Receipt",
  bodyText: "Thank you for your...",
  bodyHtml: "<html>...",
  receivedAt: Timestamp,
  processedAt: Timestamp,
  status: "pending_review",            // pending_review | approved | rejected | ignored
  attachments: [
    {
      filename: "receipt.pdf",
      mimeType: "application/pdf",
      size: 45678,
      pdfText: "Invoice\nTotal: JD 350.00\n..."
    }
  ],
  extractedData: {
    amount: 350.00,
    currency: "JOD",
    date: "2026-07-01",
    vendor: "Arduino Store",
    description: "Order #12345",
    category: "Hardware",
    confidence: 0.85
  },
  transactionId: "auto-generated",     // ref to the draft transaction
  reviewedBy: null,
  reviewedAt: null,
  createdAt: Timestamp,
}
```

### `ady_transactions` — Draft Transaction

The draft transaction created from the email:

```javascript
{
  id: "auto-generated",
  projectId: "ady",
  type: "cost",
  category: "Hardware",               // from extraction
  amount: 350.00,
  currency: "JOD",
  date: Timestamp,                    // from extraction or email date
  description: "Order #12345 — Arduino Store",
  vendor: "Arduino Store",
  fundingSource: "psut",
  evidenceAttached: true,
  evidenceFiles: [
    {
      id: "auto",
      fileName: "receipt.pdf",
      fileType: "application/pdf",
      fileSize: 45678,
      base64Data: "..."               // the PDF file (base64)
    }
  ],
  source: "email",                    // NEW FIELD
  emailInboxId: "auto-generated",     // NEW FIELD — ref to email_inbox doc
  reviewStatus: "pending_review",     // NEW FIELD — pending_review | approved | rejected
  periodId: null,                     // assigned during review
  notes: "Auto-imported from email: store@arduino.com",
  createdAt: Timestamp,
  updatedAt: Timestamp,
}
```

---

## New Transaction Fields (Already in our types — iOS needs to add)

These 3 fields are already in the web `Transaction` type and in Firestore:

| Field | Type | Values | Description |
|---|---|---|---|
| `source` | String? | `"email"`, `"manual"`, `"web"`, `"ios"` | Origin of the transaction |
| `emailInboxId` | String? | Firestore doc ID | Reference to `email_inbox` document |
| `reviewStatus` | String? | `"pending_review"`, `"approved"`, `"rejected"` | Review state |

**iOS needs to:**
1. Add these 3 optional fields to the `Transaction` Swift model
2. They're all optional so they won't break existing decoding

---

## What iOS Needs to Do

### 1. Add 3 new fields to `Transaction` model

```swift
struct Transaction: Codable, Identifiable {
    // ... existing fields ...
    
    // Email-origin fields (new)
    var source: String?              // "email", "manual", "web", "ios"
    var emailInboxId: String?        // ref to email_inbox doc
    var reviewStatus: String?        // "pending_review", "approved", "rejected"
}
```

### 2. Show pending review transactions with amber badge

In the transactions list:
- Transactions where `reviewStatus == "pending_review"` → show amber "Pending" badge
- Transactions where `source == "email"` → show email icon (envelope)
- These should still appear in the normal list, just with visual distinction

### 3. Add approve/reject actions in transaction detail

When viewing a transaction with `reviewStatus == "pending_review"`:
- Show "Approve" button → sets `reviewStatus: "approved"` and assigns `periodId`
- Show "Reject" button → sets `reviewStatus: "rejected"`
- Show "From Email" section with sender info (from `email_inbox` doc referenced by `emailInboxId`)

### 4. Add "Pending Review" filter

Add a filter option in the transactions list to show only `reviewStatus == "pending_review"`.

### 5. Show evidence (PDF receipt)

Transactions with `evidenceAttached == true` and `evidenceFiles` array:
- Show a "View Receipt" button
- Decode `base64Data` from the first evidence file
- Display PDF in a viewer (same as existing evidence viewer if you have one)

---

## How the Full Flow Works

```
1. Vendor sends email with PDF receipt to info@contact.the-rs.com
2. Resend receives the email → fires webhook to processInboundEmail
3. Cloud Function:
   a. Parses PDF text
   b. Extracts amount, date, vendor, category via regex
   c. Creates email_inbox document
   d. Creates draft transaction (reviewStatus: pending_review)
4. Web portal: Email Inbox tab shows the email with extracted data
5. Admin reviews → edits if needed → clicks "Approve & Create"
6. Transaction is finalized (reviewStatus: approved, periodId assigned)
7. iOS sees the transaction in the list:
   - Before approval: amber "Pending" badge
   - After approval: normal transaction
```

---

## Resend Setup (Already Done on Web Side)

- ✅ `RESEND_API_KEY` stored as Firebase secret
- ✅ `processInboundEmail` Cloud Function deployed
- ✅ Web UI (Email Inbox tab) built and deployed

**Still needed (Mohammad is doing this):**
- Enable Resend inbound for `contact.the-rs.com` domain
- Add MX records to DNS
- Set webhook URL to the Cloud Function URL above

---

## Testing

To test the flow once Resend inbound is configured:
1. Send an email to `info@contact.the-rs.com` with a PDF receipt attached
2. Wait a few seconds for the webhook to fire
3. Check the Email Inbox tab on the web portal
4. The email should appear with extracted data and PDF preview
5. Click it → review → approve

You can also test the webhook directly with curl:

```bash
curl -X POST https://us-central1-robotics-website-5593f.cloudfunctions.net/processInboundEmail \
  -H "Content-Type: application/json" \
  -d '{
    "from": "test@store.com",
    "from_name": "Test Store",
    "to": "info@contact.the-rs.com",
    "subject": "Test Receipt",
    "text": "Total: JD 150.00\nDate: 2026-07-03",
    "message_id": "test-001",
    "attachments": []
  }'
```

This will create an email_inbox doc and a draft transaction without a PDF.

---

## Questions?

Reply by adding a new `.md` file to `shared/docs/`.
