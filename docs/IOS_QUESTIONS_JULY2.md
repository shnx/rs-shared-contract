# iOS → Web: Questions, Comments & Requests (July 2)

**Date:** July 2, 2026
**From:** iOS team
**Re:** Questions on your updates + things we need from you

---

## 1. Achievements / timeline_entries — VERIFY DATA EXISTS

You said `timeline_entries` has 67 docs. We updated our code to read from it. But we don't trust it until we see it ourselves.

**What we did:**
- Updated `TimelineEntry` model to match your schema (entry_type, summary, amount, currency, transaction_id, payment_method, is_visible, created_by)
- Repository tries `timeline_entries` first, then falls back to `timeline` and `ady_timeline`
- Added console logging so we can see exactly how many docs each collection has

**What we need from you:**
- **Confirm the collection name is exactly `timeline_entries`** (not `timelineEntries` or `timeline-entries`)
- **Paste a sample document** (raw JSON from Firestore) so we can verify field names match
- **Confirm `date` is a string** ("2025-11-25") not a Firestore Timestamp — our FlexibleDate decoder handles both but we want to know
- **Is `is_visible` filtering needed?** Should we hide entries where `is_visible == false`? Right now we show everything.

---

## 2. Sub-collections — SKIP FOR NOW

You asked about `timeline_entry_fields`, `timeline_tags`, `timeline_links`. Our answer: **skip them**. Keep it simple. If we need tags or links later, we'll add them as fields on the main document.

---

## 3. Careers collection — WE NEED SCHEMA

We added a Careers tab that reads from `careers`. But we guessed the schema:

```
title: string
description: string
type: string (full-time, part-time, internship?)
isActive: boolean
createdAt: Date
updatedAt: Date
```

**Questions:**
- Is this correct? What are the actual field names?
- What values does `type` accept?
- Is there a `requirements` field? `location`? `salary`?
- How do job applications link to careers? Is there a `careerId` on `jobSubmissions`?

---

## 4. Documents collection — CONFIRM FIELDS

We're reading from `documents` / `ady_documents` using our existing `PortalDocument` model. It has:

```
fileName, fileUrl, category, projectId, uploadedAt, linkedTransactionId, isPDF, data
```

**Questions:**
- Does your documents schema match these fields?
- Do you write documents from the web? If so, what fields do you set?
- Is `data` used for anything? We see it in the model but not sure what it holds.

---

## 5. Contact Inquiries — CONFIRM FIELDS

We're reading from `contactInquiries`. Our model has:

```
name, email, phone, subject, message, company, body, submittedAt, createdAt, status, deviceInfo
```

**Questions:**
- Do you have a `replied` or `repliedAt` field? We want to show reply status.
- What does `status` accept? Is it just "new" / "read" / "replied"?
- Should we add a `priority` field?

---

## 6. Building Data Model — NEED TO DESIGN TOGETHER

You said we should design the full building model together. Here's our proposal:

```
Collection: buildings
  name: string
  address: string
  unitsCount: number
  occupiedUnitsCount: number
  managerUserId: string
  floorsCount: number          ← NEW
  hasElevator: boolean         ← NEW
  hasParking: boolean          ← NEW
  imageUrls: string[]          ← NEW
  totalMonthlyRent: number     ← NEW (in JOD)
  createdAt: Date
  updatedAt: Date

Sub-collection: buildings/{id}/units
  unitNumber: string
  floor: number
  monthlyRent: number
  status: "vacant" | "occupied" | "maintenance"
  tenantId: string?            ← links to tenants collection
  bedrooms: number
  bathrooms: number
  area: number                 ← in sqm

Collection: tenants
  name: string
  phone: string
  email: string?
  leaseStart: Date
  leaseEnd: Date?
  depositAmount: number
  unitId: string
  buildingId: string

Collection: rentPayments
  tenantId: string
  unitId: string
  buildingId: string
  amount: number
  dueDate: Date
  paidDate: Date?
  status: "pending" | "paid" | "late" | "partial"

Collection: maintenanceTickets
  buildingId: string
  unitId: string?
  title: string
  description: string
  priority: "low" | "medium" | "high" | "urgent"
  status: "open" | "in-progress" | "resolved"
  assignedTo: string?
  createdAt: Date
  resolvedAt: Date?
```

**Questions for you:**
- Does this match what you had in mind?
- Are you already writing to `buildings` with different fields?
- Should we start with just the main `buildings` collection and add sub-collections later?

---

## 7. CV Center / AI Extraction

You mentioned sharing your AI extraction + LaTeX generation approach. We need:
- What AI service do you use? (OpenAI, Gemini, local?)
- What's the prompt structure?
- Where is the extracted data stored? On the `jobSubmissions` document or a separate collection?
- Do you have an API endpoint we can call, or should we call the AI directly from iOS?

---

## 8. Student Portal Scope

You asked about student portal scope. We have a `student` role and a `StudentDashboardView` but it's minimal. Before we build more:

- What should students see? (Their submissions? Their CV? Job postings? Training materials?)
- Should students be able to submit assignments or just view content?
- Is this tied to a specific university (PSUT) or generic?

---

## 9. ADY Project Type — accounting or education?

You asked if ADY should be `accounting` or `education` (or dual-type). Our answer:

**ADY is `accounting`**. It's a financial management project. The "education" aspect (students, PSUT advisors) is a role-based access thing, not a project type. One project, multiple role experiences.

---

## 10. Collection Naming Convention

We noticed you're using `timeline_entries` (snake_case) while we've been using `ady_` prefix for some collections. Can we agree on a standard?

**Our proposal:**
- No prefix for shared collections (`buildings`, `careers`, `timeline_entries`, `contactInquiries`, `jobSubmissions`, `systemUsers`)
- `ady_` prefix only for ADY-specific collections (`ady_transactions`, `ady_periods`, `ady_documents`, `ady_crm`)
- All snake_case for collection names
- All camelCase for field names

Does that work for you?

---

## Summary of What We Need

| # | What | Priority |
|---|---|---|
| 1 | Confirm `timeline_entries` has data + paste a sample doc | HIGH |
| 2 | Careers collection schema | HIGH |
| 3 | Documents schema confirmation | MEDIUM |
| 4 | Contact inquiries — replied/priority fields | MEDIUM |
| 5 | Building model approval | MEDIUM |
| 6 | CV/AI extraction approach | LOW (later) |
| 7 | Student portal scope | LOW (later) |

Let us know.
