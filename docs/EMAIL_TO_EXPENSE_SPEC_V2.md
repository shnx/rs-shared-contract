# Spec: Email-to-Expense System — Updated (July 3)

**From:** iOS team
**To:** Web team
**Re:** Updated plan using existing `info@the-rs.com` email + Resend inbound webhooks. No new Gmail needed.

---

## What Already Exists

1. **`info@the-rs.com`** — sending via Resend SMTP (`smtp.resend.com:465`)
2. **Firebase Cloud Functions** with `nodemailer` — `sendCVFeedback`, `sendInvitationEmail` (deployed, working)
3. **SMTP secrets** stored as Firebase secrets: `SMTP_USER`, `SMTP_PASS`
4. **Email Test Page** at `/email-test` — mock only (logs to console)
5. **Testing Lab** at `/testing-lab` — sends real emails via Cloud Functions
6. **Resend** supports **inbound email receiving** with webhooks + attachment API

## Updated Architecture

```
Vendor sends receipt PDF
        │
        ▼
info@the-rs.com  (Resend receiving domain)
        │
        ▼  webhook: email.received event
        │
┌───────▼──────────────────────────┐
│  Firebase Cloud Function          │
│  (new: processInboundEmail)       │
│                                   │
│  1. Verify webhook signature      │
│  2. Get email content via API     │
│  3. Download PDF attachments      │
│  4. Parse PDF with pdf-parse      │
│  5. Regex extract: amount, date,  │
│     vendor, category, description │
│  6. Create email_inbox doc        │
│  7. Create draft transaction      │
└───────┬──────────────────────────┘
        │
        ▼
  Firestore: email_inbox + ady_transactions (draft)
        │
        ├──▶ Website: /email-inbox page — review & approve
        │
        └──▶ iOS: sees draft tx with "Pending Review" badge
```

## Why This Is Better Than Gmail

- **No new email account** — use existing `info@the-rs.com`
- **Real-time** — Resend sends webhook immediately when email arrives (no polling)
- **No Gmail API setup** — no OAuth, no credentials, no token refresh
- **Attachments API** — Resend gives download URLs for attachments (valid 1 hour)
- **Email content API** — Resend gives HTML + plain text body via API
- **Already have Resend account** — just need to enable receiving

## Setup Steps (Web Team)

### Step 1: Enable Resend Inbound

1. Go to **Resend Dashboard → Webhooks**
2. Click **Add Webhook**
3. URL: `https://us-central1-robotics-website-5593f.cloudfunctions.net/processInboundEmail`
4. Event type: `email.received`
5. Save

### Step 2: Verify Receiving Domain

Resend needs `the-rs.com` configured for receiving. Check:
- **Resend Dashboard → Domains** — is `the-rs.com` verified for sending?
- If yes, receiving should work automatically (same domain)
- If not, add MX records for inbound (Resend docs: https://resend.com/docs/dashboard/receiving/)

### Step 3: Add Resend API Key Secret

```bash
firebase functions:secrets:set RESEND_API_KEY
# Enter your Resend API key (re_xxxxxxxxx)
```

### Step 4: Deploy New Cloud Function

The new `processInboundEmail` function will be added to `functions/src/index.ts`.

## Cloud Function: processInboundEmail

```typescript
import { onRequest } from "firebase-functions/v2/https";
import { defineSecret } from "firebase-functions/params";
import * as admin from "firebase-admin";
import * as pdfParse from "pdf-parse";

const resendApiKey = defineSecret("RESEND_API_KEY");

export const processInboundEmail = onRequest(
  {
    region: "us-central1",
    secrets: [resendApiKey],
    maxInstances: 10,
  },
  async (req, res) => {
    // 1. Verify webhook (Resend sends a signature header)
    const event = req.body;
    if (event.type !== "email.received") {
      return res.status(200).json({ ok: true });
    }

    const { email_id, from, to, subject, attachments } = event.data;

    // 2. Check for duplicate (idempotency)
    const existing = await admin.firestore()
      .collection("email_inbox")
      .where("resendEmailId", "==", email_id)
      .get();
    if (!existing.empty) {
      return res.status(200).json({ ok: true, message: "duplicate" });
    }

    // 3. Get email content (HTML + plain text)
    const emailResp = await fetch(
      `https://api.resend.com/emails/receiving/${email_id}`,
      { headers: { Authorization: `Bearer ${resendApiKey.value()}` } }
    );
    const emailData = await emailResp.json();

    // 4. Process PDF attachments
    const processedAttachments = [];
    for (const att of attachments || []) {
      if (!att.content_type.includes("pdf")) continue;

      // Get download URL from Attachments API
      const attResp = await fetch(
        `https://api.resend.com/emails/receiving/${email_id}/attachments`,
        { headers: { Authorization: `Bearer ${resendApiKey.value()}` } }
      );
      const attData = await attResp.json();
      const found = attData.data?.find((a: any) => a.id === att.id);
      if (!found?.download_url) continue;

      // Download PDF
      const fileResp = await fetch(found.download_url);
      const buffer = Buffer.from(await fileResp.arrayBuffer());

      // Parse PDF text
      const pdfData = await pdfParse(buffer);
      const pdfText = pdfData.text;

      processedAttachments.push({
        filename: att.filename,
        mimeType: att.content_type,
        size: found.size || buffer.length,
        pdfText,
        base64Data: buffer.toString("base64"),
      });
    }

    // 5. Extract transaction data (regex, no AI)
    const combinedText = `${subject}\n${emailData.text || ""}\n${processedAttachments.map(a => a.pdfText).join("\n")}`;
    const extracted = extractTransactionData(combinedText, from);

    // 6. Create email_inbox document
    const inboxRef = await admin.firestore().collection("email_inbox").add({
      resendEmailId: email_id,
      from,
      fromName: from.split("<")[0].trim(),
      to,
      subject,
      bodyText: emailData.text || "",
      bodyHtml: emailData.html || "",
      receivedAt: admin.firestore.FieldValue.serverTimestamp(),
      processedAt: admin.firestore.FieldValue.serverTimestamp(),
      status: "pending_review",
      attachments: processedAttachments.map(a => ({
        filename: a.filename,
        mimeType: a.mimeType,
        size: a.size,
        pdfText: a.pdfText,
      })),
      extractedData: extracted,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    // 7. Create draft transaction(s) — one per PDF attachment
    for (const att of processedAttachments) {
      const txRef = await admin.firestore().collection("ady_transactions").add({
        projectId: "ady",
        type: "cost",
        category: extracted.category || "Uncategorized",
        amount: extracted.amount || 0,
        currency: extracted.currency || "JOD",
        date: extracted.date || admin.firestore.FieldValue.serverTimestamp(),
        description: extracted.description || subject,
        vendor: extracted.vendor || from.split("<")[0].trim(),
        fundingSource: "psut",
        evidenceAttached: true,
        evidenceFiles: [{
          id: admin.firestore().collection("tmp").doc().id,
          fileName: att.filename,
          fileType: att.mimeType,
          fileSize: att.size,
          base64Data: att.base64Data,
        }],
        source: "email",
        emailInboxId: inboxRef.id,
        reviewStatus: "pending_review",
        notes: `Auto-imported from email: ${from}`,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp(),
      });

      // Link transaction back to inbox
      await inboxRef.update({
        transactionId: txRef.id,
      });
    }

    // 8. If no PDF attachments, still create a draft from email body
    if (processedAttachments.length === 0 && extracted.amount) {
      const txRef = await admin.firestore().collection("ady_transactions").add({
        projectId: "ady",
        type: "cost",
        category: extracted.category || "Uncategorized",
        amount: extracted.amount,
        currency: extracted.currency || "JOD",
        date: extracted.date || admin.firestore.FieldValue.serverTimestamp(),
        description: extracted.description || subject,
        vendor: extracted.vendor || from.split("<")[0].trim(),
        fundingSource: "psut",
        evidenceAttached: false,
        source: "email",
        emailInboxId: inboxRef.id,
        reviewStatus: "pending_review",
        notes: `Auto-imported from email: ${from} (no PDF attachment)`,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp(),
      });
      await inboxRef.update({ transactionId: txRef.id });
    }

    res.status(200).json({ ok: true, processed: true });
  }
);

// ─── Regex Extraction (AI-free) ───
function extractTransactionData(text: string, fromEmail: string) {
  const result: any = {
    amount: null,
    currency: "JOD",
    date: null,
    vendor: null,
    description: null,
    category: null,
    confidence: 0,
  };

  // Amount patterns
  const amountPatterns = [
    { re: /(?:total|amount|sum|paid|balance due)[:\s]*(?:JD|JOD)?\s*([\d,]+\.?\d*)/i, currency: "JOD" },
    { re: /(?:JD|JOD)\s*([\d,]+\.?\d*)/i, currency: "JOD" },
    { re: /(?:\$|USD)\s*([\d,]+\.?\d*)/i, currency: "USD" },
    { re: /([\d,]+\.\d{2})\s*(?:JD|JOD)/i, currency: "JOD" },
  ];
  for (const p of amountPatterns) {
    const m = text.match(p.re);
    if (m) {
      result.amount = parseFloat(m[1].replace(/,/g, ""));
      result.currency = p.currency;
      result.confidence += 0.3;
      break;
    }
  }

  // Date patterns
  const datePatterns = [
    /(\d{4}-\d{2}-\d{2})/,
    /(\d{2}\/\d{2}\/\d{4})/,
    /(\d{2}-\d{2}-\d{4})/,
    /(?:date)[:\s]*(\d{1,2}\s+\w+\s+\d{4})/i,
  ];
  for (const p of datePatterns) {
    const m = text.match(p);
    if (m) {
      result.date = m[1];
      result.confidence += 0.2;
      break;
    }
  }

  // Vendor — from email sender or text
  const vendorMatch = text.match(/(?:from|vendor|merchant|seller)[:\s]*([^\n]{2,50})/i);
  result.vendor = vendorMatch ? vendorMatch[1].trim() : fromEmail.split("<")[0].trim();
  result.confidence += 0.2;

  // Category — keyword matching
  const categoryKeywords: Record<string, string[]> = {
    "Hardware": ["arduino", "sensor", "motor", "servo", "raspberry", "pi", "esp32", "pcb", "robot", "circuit", "led", "battery"],
    "Software": ["subscription", "github", "license", "software", "cloud", "aws", "google", "api", "domain", "hosting"],
    "Personnel": ["salary", "freelance", "intern", "stipend", "wage", "developer", "designer", "consultant"],
    "Operations": ["shipping", "delivery", "transport", "food", "office", "printing", "internet", "phone", "rent"],
    "Fees": ["bank", "fee", "transfer", "paypal", "stripe"],
  };
  const lower = text.toLowerCase();
  for (const [cat, keywords] of Object.entries(categoryKeywords)) {
    if (keywords.some(kw => lower.includes(kw))) {
      result.category = cat;
      result.confidence += 0.15;
      break;
    }
  }

  // Description — first meaningful line
  const lines = text.split("\n").map(l => l.trim()).filter(l => l.length > 5);
  if (lines.length > 0) {
    result.description = lines[0].substring(0, 100);
    result.confidence += 0.1;
  }

  return result;
}
```

## Website UI — Email Inbox Page

### New route: `/email-inbox` (admin only)

**List view:**
- Table of received emails: sender, subject, received date, status badge, extracted amount, confidence, attachment count
- Filter by status: All / Pending / Approved / Rejected
- "Poll now" button (manual refresh)

**Detail view (click email):**
- Email metadata: from, to, subject, date
- Email body (rendered HTML or plain text)
- PDF preview (embedded viewer)
- Extracted data fields (editable): amount, currency, date, vendor, category, description, fundingSource
- Action buttons: Approve & Create Transaction / Reject / Ignore / Re-extract

**Draft transactions in transactions list:**
- Amber "Pending Review" badge
- Email icon (envelope)
- Clicking opens review sheet

## Existing Email Pages

We already have two email-related pages:

1. **`/email-test`** (`EmailTestPage.tsx`) — mock email sender, logs to console. Can be kept for dev testing.
2. **`/testing-lab`** (`TestingLab.tsx`) — real email sender via Cloud Functions. Has "Send CV Feedback Email" and "Send Invitation Email" buttons.

The new `/email-inbox` page is for **receiving** — it's separate from the sending pages.

## Agreed Transaction Fields (Both Platforms)

Already synced. Added 3 new fields to both iOS and web:

| Field | Type | iOS | Web |
|---|---|---|---|
| `source` | String? | ✅ added | ✅ added |
| `emailInboxId` | String? | ✅ added | ✅ added |
| `reviewStatus` | String? | ✅ added | ✅ added |

Full field list in `EMAIL_TO_EXPENSE_SPEC.md` (previous version).

## What Web Team Needs to Do

1. **Enable Resend inbound** — add webhook URL in Resend dashboard
2. **Verify MX records** for `the-rs.com` receiving (Resend docs)
3. **Set `RESEND_API_KEY` secret** — `firebase functions:secrets:set RESEND_API_KEY`
4. **Install `pdf-parse` in functions/** — `cd functions && npm install pdf-parse`
5. **Add `processInboundEmail` function** to `functions/src/index.ts`
6. **Deploy functions** — `firebase deploy --only functions`
7. **Build `/email-inbox` page** — list + detail + review/approve UI
8. **Add route** in `App.tsx`

## What iOS Will Do

1. ✅ Added `source`, `emailInboxId`, `reviewStatus` to Transaction model
2. Show pending review transactions with amber badge
3. Add approve/reject actions in transaction detail
4. Add "From Email" section showing sender info

## Vendors Can Now Send To

Share `info@the-rs.com` with vendors:
- Arduino Store → send receipts to `info@the-rs.com`
- Google Cloud → forward billing emails
- Apple Developer → forward receipts
- Any vendor → ask them to CC `info@the-rs.com`

**Also:** Set up forwarding from your personal Gmail:
- Filter: if subject contains "receipt" or "invoice" → forward to `info@the-rs.com`

## Questions

1. **Is `the-rs.com` verified in Resend for sending?** If yes, receiving should work with same domain.
2. **Do we need separate MX records for receiving?** Check Resend docs — may need to add MX records pointing to Resend's inbound servers.
3. **Webhook URL** — needs to be publicly accessible. Firebase Cloud Functions v2 are public by default. OK?
4. **Multiple PDFs per email** — current plan: create one transaction per PDF. OK?
5. **Auto-approve** — should we auto-approve if confidence > 0.9? Or always manual review?

---

*Reply by adding a new `.md` file to `shared/docs/`.*
