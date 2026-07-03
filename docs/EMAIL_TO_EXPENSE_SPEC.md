# Spec: Email-to-Expense System (July 3)

**From:** iOS team
**To:** Web team
**Re:** Plan for receiving expense emails with PDF attachments, extracting data, and creating transactions automatically. Both platforms need to agree on transaction fields and stay synced.

---

## Overview

We want a dedicated email address (e.g. `receipts@robotics.psut.edu.jo` or a Gmail like `rs.receipts@gmail.com`) that receives expense emails with PDF receipt attachments. The system:

1. **Receives** the email (with subject, body, and PDF attachment)
2. **Extracts** text from the PDF (no AI needed — use `pdf-parse`)
3. **Parses** the email body + PDF text for transaction fields (amount, date, vendor, description)
4. **Creates** a draft transaction in Firestore (status: `pending_review`)
5. **Shows** the email + extracted data on the website for review/approval
6. **iOS** sees the draft transaction and can also review/approve

**AI-free approach preferred** — use regex + keyword matching for extraction. If AI is needed later, the email body + PDF text can be sent as a prompt, but let's start without it.

---

## Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Email      │────▶│  Web Server      │────▶│  Firestore      │
│  receipts@  │     │  (polls Gmail    │     │  email_inbox    │
│  gmail.com  │     │  via Gmail API)  │     │  collection     │
└─────────────┘     └────────┬─────────┘     └────────┬────────┘
                             │                        │
                     ┌───────▼────────┐       ┌───────▼────────┐
                     │  PDF Parser    │       │  ady_transactions│
                     │  (pdf-parse)   │       │  (status: draft) │
                     │  + Regex       │       └───────┬────────┘
                     └────────────────┘               │
                                               ┌───────▼────────┐
                                               │  Website UI    │
                                               │  Email Inbox    │
                                               │  Review & Approve│
                                               └───────┬────────┘
                                                       │
                                               ┌───────▼────────┐
                                               │  iOS App       │
                                               │  Sees draft tx  │
                                               │  Review & Approve│
                                               └────────────────┘
```

---

## Email Setup

### Option A: Gmail + Gmail API (Recommended — Free, No Cost)

1. Create a Gmail account: `rs.receipts@gmail.com`
2. Enable Gmail API in Google Cloud Console
3. Use the existing `googleapis` package (already installed in server)
4. Server polls Gmail every 5 minutes for new emails
5. Download attachments, parse PDFs, create draft transactions

**Pros:** Free, no SMTP server needed, Gmail API is reliable
**Cons:** Polling (not real-time), but 5-min interval is fine

### Option B: Custom Email + IMAP

1. Set up `receipts@robotics.psut.edu.jo` via Google Workspace or Zoho
2. Use IMAP to poll the inbox
3. Requires `imap` + `mailparser` npm packages

**Pros:** Professional domain
**Cons:** Requires email hosting, more setup

**Recommendation:** Start with Option A (Gmail). Can migrate to Option B later.

---

## Gmail API Polling Flow

```
1. Authenticate with Gmail API (OAuth2 or Service Account)
2. List messages with label "INBOX", query "has:attachment filename:pdf newer_than:1d"
3. For each new message:
   a. Get full message (metadata + payload)
   b. Extract subject, from, date, body text
   c. Find PDF attachment parts (mimeType: "application/pdf")
   d. Download attachment data (base64)
   e. Parse PDF with pdf-parse → extract text
   f. Run regex extraction on (email body + PDF text)
   g. Create email_inbox document in Firestore
   h. Create draft transaction in ady_transactions
   i. Mark Gmail message as read / add label "processed"
```

---

## Firestore Collections

### `email_inbox` (NEW collection)

```javascript
{
  id: "auto-generated",
  gmailMessageId: "18c3f...",        // Gmail message ID for dedup
  from: "store@arduino.com",          // sender email
  fromName: "Arduino Store",          // sender display name
  to: "rs.receipts@gmail.com",
  subject: "Order #12345 — Receipt",
  bodyText: "Thank you for your...",  // plain text body
  bodyHtml: "<html>...",              // HTML body (optional)
  receivedAt: Timestamp,              // email date
  processedAt: Timestamp,             // when server processed it
  status: "pending_review",           // pending_review | approved | rejected | ignored
  attachments: [
    {
      filename: "receipt_12345.pdf",
      mimeType: "application/pdf",
      size: 45678,
      pdfText: "Invoice\nTotal: JD 350.00\nDate: 2026-07-01\n...",  // extracted text
    }
  ],
  extractedData: {
    amount: 350.00,
    currency: "JOD",
    date: "2026-07-01",
    vendor: "Arduino Store",
    description: "Order #12345",
    category: "Hardware",             // guessed from keywords
    fundingSource: "psut",            // default
    confidence: 0.85,                 // how confident the extraction is
    method: "regex",                  // "regex" or "ai"
  },
  transactionId: "auto-generated",    // ID of the draft transaction created
  reviewedBy: null,                   // user who reviewed (set on approve/reject)
  reviewedAt: null,
  createdAt: Timestamp,
}
```

### `ady_transactions` — Draft Transaction

The draft transaction created from the email:

```javascript
{
  id: "email_<gmailMessageId>",
  projectId: "ady",
  type: "cost",                       // default to expense
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
      fileName: "receipt_12345.pdf",
      fileType: "application/pdf",
      fileSize: 45678,
      base64Data: "...",              // the PDF file
    }
  ],
  // NEW FIELDS for email-origin transactions:
  source: "email",                    // "email" | "manual" | "web" | "ios"
  emailInboxId: "auto-generated",     // ref to email_inbox doc
  reviewStatus: "pending_review",     // "pending_review" | "approved" | "rejected"
  // Standard fields:
  periodId: null,                     // assigned during review
  notes: "Auto-imported from email: receipts@...",
  createdAt: Timestamp,
  updatedAt: Timestamp,
}
```

---

## New Transaction Fields (Both Platforms Must Sync)

### Fields to add to `Transaction` model

| Field | Type | iOS | Web | Description |
|---|---|---|---|---|
| `source` | String? | ✅ add | ✅ add | Origin: "email", "manual", "web", "ios" |
| `emailInboxId` | String? | ✅ add | ✅ add | Reference to `email_inbox` document |
| `reviewStatus` | String? | ✅ add | ✅ add | "pending_review", "approved", "rejected" |

### Full Agreed Transaction Fields (Both Platforms)

| Field | Type | Required | iOS | Web | Notes |
|---|---|---|---|---|---|
| `id` | String | auto | ✅ | ✅ | Firestore doc ID |
| `projectId` | String? | no | ✅ | ✅ | "ady" for ADY project |
| `storeId` | String? | no | ✅ | ✅ | Legacy |
| `type` | String | yes | ✅ | ✅ | "cost", "bill", "revenue", "payment", "check", "freelance", "salary" |
| `category` | String? | no | ✅ | ✅ | e.g. "Hardware", "Software" |
| `amount` | Decimal/Number | yes | ✅ | ✅ | In original currency |
| `currency` | String | yes | ✅ | ✅ | "JOD", "USD", "EUR" (EUR treated as JOD) |
| `date` | Date/Timestamp | yes | ✅ | ✅ | Transaction date |
| `description` | String? | no | ✅ | ✅ | Human-readable description |
| `note` / `notes` | String? | no | ✅ | ✅ | Internal notes |
| `periodId` | String? | no | ✅ | ✅ | Assigned period |
| `vendor` | String? | no | ✅ | ✅ | Who we paid / received from |
| `fundingSource` | String? | no | ✅ | ✅ | "psut", "own-capital", etc. |
| `evidenceAttached` | Bool? | no | ✅ | ✅ | Has evidence files |
| `evidenceFiles` | [EvidenceFile]? | no | ✅ | ✅ | Base64-encoded files |
| `attachmentUrl` | String? | no | ✅ | ✅ | Legacy URL-based attachment |
| `installmentId` | String? | no | ✅ | ✅ | Linked installment |
| `subscriptionId` | String? | no | ✅ | ✅ | Linked subscription |
| `createdAt` | Date? | auto | ✅ | ✅ | Creation timestamp |
| `updatedAt` | Date? | auto | ✅ | ✅ | Last update timestamp |
| **`source`** | String? | **NEW** | **add** | **add** | "email", "manual", "web", "ios" |
| **`emailInboxId`** | String? | **NEW** | **add** | **add** | Ref to email_inbox doc |
| **`reviewStatus`** | String? | **NEW** | **add** | **add** | "pending_review", "approved", "rejected" |
| **`calculationType`** | String? | no | ❌ | ✅ | "fixed" or "hourly" (web only) |
| **`hourlyRate`** | Number? | no | ❌ | ✅ | Web only |
| **`hoursWorked`** | Number? | no | ❌ | ✅ | Web only |

**iOS note:** `calculationType`, `hourlyRate`, `hoursWorked` are web-only fields. iOS can ignore them. They won't break iOS decoding since all fields are optional.

---

## PDF Text Extraction (AI-Free)

Using `pdf-parse` (already installed):

```javascript
const pdfParse = require('pdf-parse');

async function extractPdfText(base64Data) {
  const buffer = Buffer.from(base64Data, 'base64');
  const data = await pdfParse(buffer);
  return data.text;  // raw text from all pages
}
```

## Regex Extraction (AI-Free)

Extract transaction fields from combined email body + PDF text:

```javascript
function extractTransactionData(text) {
  const result = {
    amount: null,
    currency: 'JOD',
    date: null,
    vendor: null,
    description: null,
    category: null,
    confidence: 0,
  };

  // Amount — look for "JD", "JOD", "$", "USD", "EUR" followed by number
  const amountPatterns = [
    /(?:total|amount|sum|paid|balance due)[:\s]*(?:JD|JOD)?\s*([\d,]+\.?\d*)/i,
    /(?:JD|JOD)\s*([\d,]+\.?\d*)/i,
    /(?:\$|USD)\s*([\d,]+\.?\d*)/i,
    /([\d,]+\.\d{2})\s*(?:JD|JOD)/i,
  ];
  for (const pattern of amountPatterns) {
    const match = text.match(pattern);
    if (match) {
      result.amount = parseFloat(match[1].replace(/,/g, ''));
      if (pattern.source.includes('USD|\\$')) result.currency = 'USD';
      if (pattern.source.includes('EUR')) result.currency = 'EUR';
      result.confidence += 0.3;
      break;
    }
  }

  // Date — look for common date formats
  const datePatterns = [
    /(\d{4}-\d{2}-\d{2})/,
    /(\d{2}\/\d{2}\/\d{4})/,
    /(\d{2}-\d{2}-\d{4})/,
    /(?:date)[:\s]*(\d{1,2}\s+\w+\s+\d{4})/i,
  ];
  for (const pattern of datePatterns) {
    const match = text.match(pattern);
    if (match) {
      result.date = match[1];
      result.confidence += 0.2;
      break;
    }
  }

  // Vendor — look for "from", "vendor", "merchant", or use email sender
  const vendorMatch = text.match(/(?:from|vendor|merchant|seller)[:\s]*([^\n]{2,50})/i);
  if (vendorMatch) {
    result.vendor = vendorMatch[1].trim();
    result.confidence += 0.2;
  }

  // Category — keyword matching
  const categoryKeywords = {
    'Hardware': ['arduino', 'sensor', 'motor', 'servo', 'raspberry', 'pi', 'esp32', 'pcb', 'robot', 'circuit', 'led', 'battery'],
    'Software': ['subscription', 'github', 'license', 'software', 'cloud', 'aws', 'google', 'api', 'domain', 'hosting'],
    'Personnel': ['salary', 'freelance', 'intern', 'stipend', 'wage', 'developer', 'designer', 'consultant'],
    'Operations': ['shipping', 'delivery', 'transport', 'food', 'office', 'printing', 'internet', 'phone', 'rent'],
    'Fees': ['bank', 'fee', 'transfer', 'paypal', 'stripe'],
  };
  const lowerText = text.toLowerCase();
  for (const [cat, keywords] of Object.entries(categoryKeywords)) {
    if (keywords.some(kw => lowerText.includes(kw))) {
      result.category = cat;
      result.confidence += 0.15;
      break;
    }
  }

  // Description — use subject line or first meaningful line
  const lines = text.split('\n').map(l => l.trim()).filter(l => l.length > 5);
  if (lines.length > 0) {
    result.description = lines[0].substring(0, 100);
    result.confidence += 0.1;
  }

  return result;
}
```

### If AI is needed later (optional)

```javascript
// Send email body + PDF text to Gemini/OpenAI for extraction
const prompt = `
You are a receipt parser. Extract transaction details from this email and PDF text.
Return JSON with: amount, currency (JOD/USD/EUR), date (YYYY-MM-DD), vendor, description, category (Hardware/Software/Personnel/Operations/Fees/Funding).

Email subject: ${emailSubject}
Email body: ${emailBody}
PDF text: ${pdfText}

Return only valid JSON, no explanation.
`;
```

---

## Website UI — Email Inbox Page

### New page: `/email-inbox` (admin only)

Shows list of received emails with:
- **Sender** (from name + email)
- **Subject**
- **Received date**
- **Status badge**: Pending (amber), Approved (green), Rejected (red), Ignored (gray)
- **Extracted amount** (if found)
- **Confidence score** (color-coded)
- **Attachment count**

### Email Detail View (click to open)

Shows:
1. **Email metadata**: from, to, subject, date, body text
2. **PDF preview**: embedded PDF viewer (base64 → blob URL)
3. **Extracted data**: amount, currency, date, vendor, category, description — **editable fields**
4. **Action buttons**:
   - **Approve & Create Transaction** — creates the transaction with edited data
   - **Reject** — marks as rejected, no transaction created
   - **Ignore** — marks as ignored (e.g., spam)
   - **Re-extract** — runs extraction again (useful after editing)

### Draft Transactions in Transactions List

Transactions with `reviewStatus: "pending_review"` show with:
- Amber badge "Pending Review"
- Email icon
- Clicking opens the review sheet with PDF preview

---

## iOS — Email-Origin Transactions

### What iOS sees

iOS loads transactions from `ady_transactions` as usual. Draft transactions (`reviewStatus: "pending_review"`) appear in the list with:
- Amber "Pending" badge
- Email icon (envelope)
- Tapping shows the transaction detail with evidence (the PDF)

### iOS Transaction Model Changes

Add 3 new optional fields to `Transaction`:

```swift
struct Transaction: Codable, Identifiable {
    // ... existing fields ...

    // Email-origin fields (new)
    var source: String?              // "email", "manual", "web", "ios"
    var emailInboxId: String?        // ref to email_inbox doc
    var reviewStatus: String?        // "pending_review", "approved", "rejected"
}
```

### iOS UI Changes

1. **Transactions list**: Show pending review transactions with amber badge
2. **Transaction detail**: Show "From Email" section with sender info
3. **Filter**: Add "Pending Review" filter option
4. **Approve action**: Button to set `reviewStatus: "approved"` and assign to period

---

## Server Implementation Plan

### New files

```
server/
  emailInbox.js          — Gmail API polling + PDF parsing + extraction
  emailInboxRoutes.js    — Express routes for email inbox UI
```

### New routes

```
GET  /api/v1/email-inbox              — list emails (paginated, filterable)
GET  /api/v1/email-inbox/:id          — get email detail + extracted data
POST /api/v1/email-inbox/:id/approve  — approve & finalize transaction
POST /api/v1/email-inbox/:id/reject   — reject (no transaction)
POST /api/v1/email-inbox/:id/ignore   — ignore (spam)
POST /api/v1/email-inbox/:id/reextract — re-run extraction
POST /api/v1/email-inbox/poll         — manually trigger Gmail poll
GET  /api/v1/email-inbox/stats        — counts: pending, approved, rejected
```

### Cron / polling

```javascript
// Poll Gmail every 5 minutes
setInterval(async () => {
  try {
    await pollGmailInbox();
  } catch (err) {
    console.error('Gmail poll error:', err);
  }
}, 5 * 60 * 1000);
```

---

## Setup Steps

### 1. Gmail Account Setup
- Create `rs.receipts@gmail.com`
- Enable Gmail API in Google Cloud Console
- Create OAuth2 credentials (or use service account)
- Store credentials in `server/credentials/gmail-credentials.json`

### 2. Server Dependencies
```bash
cd server && npm install googleapis pdf-parse
# googleapis already installed ✅
# pdf-parse already in root package.json ✅
```

### 3. Firestore Security Rules
```
match /email_inbox/{doc} {
  allow read: if isAdmin();
  allow write: if isAdmin() || isServer();
}
```

### 4. Web UI
- Add "Email Inbox" tab or page (admin only)
- Add draft transaction badges to transactions list
- Add PDF preview component

### 5. iOS
- Add 3 new fields to `Transaction` model
- Show pending review badge in transactions list
- Add approve/reject actions

---

## Vendors Can Send To

Share the email address with vendors:
- Arduino Store → `receipts@rs.com`
- Google Cloud → billing emails forwarded to `rs.receipts@gmail.com`
- Apple Developer → receipts forwarded
- Any vendor → ask them to CC `rs.receipts@gmail.com` on invoices

**Also:** Set up email forwarding rules:
- `mohammedshannak@gmail.com` → forward receipts/invoices to `rs.receipts@gmail.com`
- Or use Gmail filter: if subject contains "receipt" or "invoice" → forward

---

## What We Need From Web Team

1. **Confirm the transaction fields** — review the field table above, add any web-only fields we missed
2. **Set up the Gmail account** — create `rs.receipts@gmail.com`, enable Gmail API
3. **Build the server-side polling** — `emailInbox.js` with Gmail API + pdf-parse
4. **Build the web UI** — email inbox page with review/approve flow
5. **Add `source`, `emailInboxId`, `reviewStatus` to your Transaction type**
6. **Tell us the Gmail address** once created so we can share it with vendors

## What iOS Will Do

1. **Add 3 new fields** to `Transaction` model (`source`, `emailInboxId`, `reviewStatus`)
2. **Show pending review transactions** with amber badge in transactions list
3. **Add approve/reject actions** in transaction detail
4. **Add "From Email" section** showing sender info and PDF preview

---

## Questions for Both Teams

1. **Gmail vs custom domain?** We prefer Gmail (free, easy API). OK?
2. **Polling interval?** 5 minutes OK? Or do we need real-time?
3. **Auto-approve threshold?** Should we auto-approve if confidence > 0.9? Or always require manual review?
4. **Multiple PDFs?** What if an email has 2+ PDF attachments — create one transaction per PDF?
5. **Non-PDF attachments?** Images (JPEG/PNG) — should we OCR them too? Or PDF only for now?
6. **Email forwarding?** Should we forward existing receipts from personal email to the receipts Gmail?

---

*Reply by adding a new `.md` file to `shared/docs/`.*
