# Web → iOS: Answers to All 10 Questions (July 2, Night)

**Date:** July 2, 2026, 10:25pm UTC+2
**From:** Web team
**Re:** Answering everything in IOS_QUESTIONS_JULY2.md + IOS_STATUS_LATE_JULY2.md

---

## 1. timeline_entries — CONFIRMED + Sample Doc ✅

**Collection name is exactly `timeline_entries`** (snake_case, no prefix).

**`date` is a string** — format is `"YYYY-MM-DD"` (e.g. `"2025-11-25"`). Not a Firestore Timestamp.

**`is_visible` filtering:** Yes, you should hide entries where `is_visible == false`. We filter on this in our UI.

**Raw sample document from Firestore:**
```json
{
  "id": "1764516544280a24b8luir",
  "title": "Pro plan service subscription time",
  "summary": "Category: Subscriptions\nSource: own-capital\nInvoice: ",
  "date": "2025-12-01",
  "entry_type": "expense",
  "is_visible": true,
  "amount": 25,
  "currency": "JOD",
  "category": "Subscriptions",
  "payment_method": "own-capital",
  "transaction_id": "1764516544280a24b8luir",
  "project_id": "1764173848951kdyjg245d",
  "created_by": "shannak",
  "created_at": "2025-12-01T14:22:24.280Z",
  "updated_at": "2025-12-01T14:22:24.280Z"
}
```

**Another sample (milestone type, no amount):**
```json
{
  "id": "abc123milestone001",
  "title": "ADY Project Launched",
  "summary": "Official project kickoff with initial team and budget allocation.",
  "date": "2025-11-20",
  "entry_type": "milestone",
  "is_visible": true,
  "project_id": "1764173848951kdyjg245d",
  "created_by": "shannak",
  "created_at": "2025-11-20T10:00:00.000Z"
}
```

**Breakdown of 67 docs:**
- 57 `expense` entries (auto-created from transactions, have `amount`, `currency`, `transaction_id`)
- 3 `income` entries (auto-created from revenue transactions)
- 7 `milestone` entries (manually created, no amount)

---

## 2. Sub-collections — SKIP ✅

Agreed. No sub-collections for timeline entries. Keep everything on the main document.

---

## 3. Careers Collection — FULL SCHEMA ✅

Our `CareerOpportunity` interface:

```typescript
interface CareerOpportunity {
  id: string;
  title: string;
  titleAr?: string;          // Arabic title (bilingual)
  department: string;        // e.g. "Engineering", "Marketing"
  location: string;          // e.g. "Amman, Jordan" or "Remote"
  type: JobType;             // 'full-time' | 'part-time' | 'freelance' | 'internship' | 'course'
  description: string;
  descriptionAr?: string;    // Arabic description
  isActive: boolean;         // whether the job is currently accepting applications
  createdAt: Date;
  updatedAt?: Date;
  aiFormatted?: boolean;     // whether AI helped format the listing
}
```

**JobType values:** `'full-time' | 'part-time' | 'freelance' | 'internship' | 'course'`

**No `requirements`, `salary`, or `requirements` field** currently. We kept it simple. We can add these later if needed.

**Job applications link:** Yes, `JobSubmission` has a `careerId` field that links to `CareerOpportunity.id`:

```typescript
interface JobSubmission {
  id: string;
  // ... candidate details ...
  careerId?: string;         // links to a CareerOpportunity
  isGraduate?: boolean;
  jobType?: JobType;
  acceptsUnpaidInternship?: boolean;
  // ...
}
```

**Collection name:** We use `career_opportunities` in Firestore (via `STORAGE_KEYS.CAREER_OPPORTUNITIES = 'career_opportunities'`). What collection name are you reading from? If you're using `careers`, we may need to align.

---

## 4. Documents Collection — FULL SCHEMA ✅

Our `PortalDocument` interface:

```typescript
interface PortalDocument {
  id: string;
  projectId: string;
  fileName: string;
  fileType: string;          // MIME type: "application/pdf", "image/png", etc.
  fileSize: number;          // in bytes
  category: 'invoice' | 'contract' | 'receipt' | 'report' | 'other';
  base64Data: string;        // base64-encoded file content
  extractedText?: string;    // OCR/AI extracted text from the document
  extracted?: {
    amount?: number;
    currency?: string;
    date?: string;
    vendor?: string;
    invoiceNumber?: string;
  };
  linkedTransactionId?: string;  // links to a Transaction if this doc is evidence
  uploadedBy: string;            // user ID
  uploadedAt: Date;
}
```

**Differences from your model:**
- We don't have `fileUrl` — we store `base64Data` directly (file content encoded)
- We don't have `isPDF` — we use `fileType` (MIME type) which covers all formats
- We don't have `data` as a generic field — we have `extractedText` and `extracted` (structured)
- We have `category` with 5 values: `invoice`, `contract`, `receipt`, `report`, `other`
- We have `linkedTransactionId` to link documents as evidence for transactions

**Collection name:** We use `documents` in Firestore.

---

## 5. Contact Inquiries — FULL SCHEMA ✅

Our `ContactSubmission` interface:

```typescript
interface ContactSubmission {
  id: string;
  name: string;
  email: string;
  company?: string;          // optional
  message: string;           // the message body
  status: 'new' | 'replied' | 'archived';
  notes?: string;            // internal notes (not visible to submitter)
  submittedAt: Date;
}
```

**Answers to your questions:**
- **`replied` / `repliedAt`:** We don't have a `repliedAt` timestamp. We use `status: 'replied'` to mark it as replied. We could add `repliedAt` if you need it.
- **`status` values:** `'new' | 'replied' | 'archived'` — that's it. No "read" state.
- **`priority`:** We don't have it. We could add it if you want.
- **No `subject` field** — our contact form is simple: name, email, company, message.
- **No `phone` field** — just email.
- **No `body` field** — we use `message`.

**Collection name:** We use `contact_submissions` in Firestore (via `STORAGE_KEYS.CONTACT_SUBMISSIONS = 'contact_submissions'`). You mentioned `contactInquiries` — we need to align this.

---

## 6. Building Data Model — APPROVED WITH NOTES ✅

Your proposal is great. Our current `Building` interface is simpler:

```typescript
interface Building {
  id: string;
  name: string;
  address: string;
  unitsCount: number;
  occupiedUnitsCount: number;
  managerUserId?: string;
  createdAt?: Date;
  updatedAt?: Date;
}
```

**Your additions we approve:**
- ✅ `floorsCount` — add it
- ✅ `hasElevator` — add it
- ✅ `hasParking` — add it
- ✅ `imageUrls: string[]` — add it
- ✅ `totalMonthlyRent` — add it (in JOD)

**Sub-collections:** Let's start with just the main `buildings` collection and add sub-collections (units, tenants, rentPayments, maintenanceTickets) later. We agree with your schema design for when we do.

**For now:** Update the `Building` interface to include your new fields, and we'll both read/write the same schema.

---

## 7. CV Center / AI Extraction — ANSWERED

**AI service:** We use **Google Gemini** (via Firebase Functions) for CV text extraction and LaTeX generation.

**Flow:**
1. Candidate uploads CV (PDF) → stored as `base64Data` on `JobSubmission`
2. Admin clicks "Prepare CV" → Cloud Function sends PDF to Gemini for extraction
3. Gemini returns structured data: name, email, phone, skills, experience, education
4. Extracted data stored on `JobSubmission.extractedData` (a `CVExtractedData` object)
5. Admin can edit, then generate LaTeX CV using another Gemini call

**Where data is stored:** On the `JobSubmission` document itself:
- `extractedText?: string` — raw OCR text
- `extractedData?: CVExtractedData` — structured extraction

**No API endpoint for iOS** — you should call Gemini directly from iOS using the Gemini SDK, or we can create a Cloud Function you can call. Let us know which you prefer.

**Prompt structure:** We'll share the exact prompts in a separate doc when we build the CV Center admin view. For now, the key is: send the PDF text to Gemini with a prompt asking for structured JSON output (name, email, phone, skills[], experience[], education[]).

---

## 8. Student Portal Scope — ANSWERED

**What students should see:**
- Their job submissions and status
- Their CV (if prepared by admin)
- Job postings / career opportunities
- Courses they're enrolled in
- Quiz results

**What students should NOT do:**
- Submit assignments (we don't have this feature)
- Edit their CV (admin does this)

**Tied to PSUT?** No, it's generic. The `PSUTCompliance` page was for a specific university requirement but is now dead code. The student portal is for any student candidate.

**Keep it simple for now:** Student sees their submissions, CV, and available jobs. We can expand later.

---

## 9. ADY Project Type — AGREED ✅

**ADY is `accounting`**. We already set this in Firestore. Agreed — the "education" aspect is role-based, not project-type based. One project, multiple role experiences.

---

## 10. Collection Naming Convention — AGREED (with alignment needed) ✅

Your proposal:
- No prefix for shared collections
- `ady_` prefix for ADY-specific collections
- snake_case for collection names
- camelCase for field names

**We agree.** But we need to align on a few collection names where we differ:

| Our collection | Your collection | Action needed |
|---|---|---|
| `contact_submissions` | `contactInquiries` | **Align** — which name? We suggest `contact_submissions` (snake_case per convention) |
| `career_opportunities` | `careers` | **Align** — we suggest `career_opportunities` (more descriptive) |
| `job_submissions` | `jobSubmissions` | **Align** — should be `job_submissions` (snake_case) |
| `ady_crm` | ? | We use `ady_crm` — is this what you read? |
| `documents` | `documents` / `ady_documents` | ✅ Aligned |

**Our current Firestore collections:**
```
projects, ady_transactions, ady_periods, ady_installments, ady_crm,
buildings, timeline_entries, documents, job_submissions, contact_submissions,
career_opportunities, ady_news, widget_configs, audit_log, system_users
```

Let's finalize the naming in the next round and both update our code.

---

## Summary of What We Need from iOS Next

| # | What | Priority |
|---|---|---|
| 1 | Confirm which collection names you're reading from (careers vs career_opportunities, contactInquiries vs contact_submissions) | HIGH |
| 2 | Confirm you're filtering `is_visible == true` on timeline_entries | MEDIUM |
| 3 | Do you want a `sendContactReply` Cloud Function? We can create one | LOW |
| 4 | Gemini SDK on iOS or Cloud Function wrapper? For CV extraction | LOW |

---

All changes deployed. Please pull `shared/` for latest docs.
