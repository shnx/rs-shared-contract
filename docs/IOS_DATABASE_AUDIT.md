# iOS Database Audit — What's Missing, What Makes Sense, What's Unused

**Date:** July 2, 2026
**From:** iOS team
**Re:** Full audit of our Firestore data models — gaps, improvements, dead fields

---

## 1. Collections We Use vs Web Has

### Collections iOS reads/writes:
| Collection | iOS Reads | iOS Writes | Notes |
|---|---|---|---|
| `systemUsers` | ✅ | ✅ | Auth, roles, permissions |
| `ady_transactions` / `transactions` | ✅ | ✅ | Merged by ID |
| `ady_periods` / `periods` | ✅ | ✅ | Merged by ID |
| `ady_installments` | ✅ | ❌ | Read-only, never written by iOS |
| `ady_stores` | ✅ | ❌ | Read-only on iOS |
| `ady_subscriptions` | ✅ | ❌ | Read-only on iOS |
| `ady_contracts` | ✅ | ❌ | Read-only on iOS |
| `ady_crm` | ✅ | ✅ | Full CRUD |
| `ady_documents` / `documents` | ✅ | ❌ | Read-only, no upload UI |
| `ady_workSessions` | ✅ | ✅ | Time tracking |
| `ady_employeePermissions` | ✅ | ✅ | Per-employee config |
| `periodSummaries` | ✅ | ✅ | iOS writes computed snapshots |
| `ady_pdf-meta` | ✅ | ✅ | OCR results |
| `timeline` / `ady_timeline` | ✅ | ❌ | Read-only |
| `buildings` | ✅ | ❌ | Read-only, scaffolding only |
| `sharedViews` | ✅ | ✅ | Shareable links |
| `jobSubmissions` | ✅ | ❌ | Read-only (web form writes) |
| `contactInquiries` | ✅ | ❌ | Read-only (web form writes) |
| `projects` | ✅ | ❌ | Read-only |
| `careers` | ❌ | ❌ | **Not used at all on iOS** |

### Collections the web has that iOS doesn't touch:
| Collection | Status | Should iOS use it? |
|---|---|---|
| `careers` | Not used | **YES** — need to add Careers tab |
| `courses` | Not used | Maybe — depends on student portal |
| `memberships` | Not used | Maybe — depends on student portal |
| `videos` / `videoTutorials` | Not used | Maybe — low priority |
| `videoAssignments` | Not used | Maybe — depends on student portal |
| `agentLog` | Not used | No — debug tool, web-only |
| `comments` | Not used | **YES** — we have a Comment model but never read/write it |
| `projectStages` | Not used | Maybe — project detail view |
| `projectMembers` | Not used | Maybe — team management |

---

## 2. Models We Have But Never Use (Dead Code)

### `Store` — read-only, no UI to manage
We have a full `Store` model with `StoreStatus` enum, but:
- No "Add Store" UI
- No "Edit Store" UI
- Only displayed as a list in some views
- **Verdict:** Keep for reading, but the model has fields we never show: `category`, `location`, `contactPerson`, `notes`

### `Subscription` — read-only, no UI
Full model with `plan`, `startDate`, `endDate`, `amount`, `currency`, `notes`. Never displayed in any view.
- **Verdict:** Either build a Subscriptions view or remove the model

### `Installment` — read-only, no UI
Model with `InstallmentStatus` (pending/paid/overdue). Never displayed.
- **Verdict:** Should be shown in Budget/Period detail view

### `Contract` — read-only, no UI
Full model with `ContractStatus` (draft/active/completed/cancelled), `terms`, `totalAmount`. Never displayed.
- **Verdict:** Should add a Contracts view or at least show in CRM detail

### `Service` — never used anywhere
Model with `title`, `description`, `price`, `currency`. No collection, no view, no reference.
- **Verdict:** Dead code — remove or build a services catalog

### `AppTheme` — never used
Model with `primaryColor`, `accentColor`, `logoUrl`. No collection, no view.
- **Verdict:** Dead code — we use hardcoded `AppThemeColors` in Theme.swift instead

### `ProjectFeature` — never used
Model for project features. No collection, no view.
- **Verdict:** Dead code or future feature

### `ProjectPayment` — never used
Model for project payments. No collection, no view.
- **Verdict:** Dead code — we use `Transaction` instead

### `ProjectDoc` — never used
Model for project documents. No collection, no view. We use `PortalDocument` instead.
- **Verdict:** Dead code — remove

### `TeamMember` — never used
Model for project team. No collection, no view.
- **Verdict:** Dead code or future feature

---

## 3. Fields We Have But Never Display

### `SystemUser`
- `projectIds: [String]?` — never read, never filtered by
- `passwordHash: String?` — legacy, should be removed (we use Firebase Auth)
- `password: String?` — legacy migration only, should be cleaned up after migration
- `hasPassword: Bool?` — set during migration but never checked in UI

### `Transaction`
- `storeId: String?` — legacy, never shown
- `subscriptionId: String?` — never shown
- `installmentId: String?` — never shown
- `attachmentUrl: String?` — never shown (we use `evidenceFiles` instead)
- `evidenceAttached: Bool?` — never checked in UI

### `CRMLead`
- `ownerUserId: String?` — set but never filtered by in UI
- `value: Double?` — never displayed
- `source: String?` — never displayed

### `JobSubmission`
- `extractedText: String?` — we show `extractedData` and `aiAnalysis` but not raw `extractedText`
- `preparedCVIds: [String]?` — added to model but no UI uses it yet

### `ContactInquiry`
- `status: String?` — never displayed or changed
- `company: String?` — displayed in detail view but not in list

---

## 4. What's Missing — Fields We Should Add

### `SystemUser` — missing fields
- `avatarUrl: String?` — profile picture URL (for display)
- `lastLoginAt: Date?` — track last login for admin dashboard
- `loginMethod: String?` — "password" | "apple" | "google" (for admin to see how users auth)
- `studentId: String?` — for student role, link to student records
- `graduationYear: Int?` — for student/advisor roles
- `university: String?` — for student/advisor roles

### `Building` — missing fields (for real estate management)
- `floorsCount: Int` — number of floors
- `elevator: Bool` — has elevator
- `parking: Bool` — has parking
- `imageUrls: [String]` — building photos
- `totalRentAmount: Decimal` — sum of all unit rents
- `managerName: String?` — denormalized manager name (avoid extra query)

### `CRMLead` — missing fields
- `estimatedCloseDate: Date?` — when do we expect to close
- `tags: [String]` — flexible tagging
- `lastContactedAt: Date?` — track engagement
- `assignedTo: String?` — different from ownerUserId (owner created, assigned is working it)

### `Transaction` — missing fields
- `receiptNumber: String?` — for paper trail
- `paymentMethod: String?` — "cash" | "bank-transfer" | "cheque" | "card"
- `approvedBy: String?` — who approved this transaction
- `approvedAt: Date?` — when approved

### `Period` — missing fields
- `carryOverFromPrevious: Decimal?` — debt/surplus from prior period
- `isLocked: Bool` — prevent edits after closing

### `JobSubmission` — missing fields
- `interviewDate: Date?` — if invited for interview
- `interviewNotes: String?` — feedback after interview
- `rejectionReason: String?` — if rejected
- `hiredAt: Date?` — if hired
- `hiredRole: String?` — actual role hired for

### `ContactInquiry` — missing fields
- `repliedAt: Date?` — when was it replied to
- `repliedBy: String?` — who replied
- `replyMessage: String?` — what was sent
- `priority: String?` — "low" | "normal" | "high"

---

## 5. Collections We Should Add

### `units` (subcollection of buildings)
```
buildings/{buildingId}/units/{unitId}
  - unitNumber: String
  - floor: Int
  - area: Double? (sqm)
  - monthlyRent: Decimal?
  - status: "vacant" | "occupied" | "maintenance"
  - tenantId: String?
  - leaseStart: Date?
  - leaseEnd: Date?
  - notes: String?
```

### `tenants`
```
tenants/{tenantId}
  - name: String
  - phone: String?
  - email: String?
  - nationalId: String?
  - buildingId: String?
  - unitId: String?
  - leaseStart: Date?
  - leaseEnd: Date?
  - depositAmount: Decimal?
  - monthlyRent: Decimal?
  - status: "active" | "inactive" | "notice"
  - createdAt: Date?
  - updatedAt: Date?
```

### `rentPayments`
```
rentPayments/{paymentId}
  - tenantId: String
  - buildingId: String
  - unitId: String
  - amount: Decimal
  - currency: Currency
  - dueDate: Date
  - paidDate: Date?
  - status: "pending" | "paid" | "overdue" | "partial"
  - method: String? ("cash" | "transfer" | "cheque")
  - notes: String?
  - createdAt: Date?
```

### `maintenanceTickets`
```
maintenanceTickets/{ticketId}
  - buildingId: String
  - unitId: String?
  - tenantId: String?
  - title: String
  - description: String
  - priority: "low" | "medium" | "high" | "urgent"
  - status: "open" | "in-progress" | "resolved" | "closed"
  - assignedTo: String?
  - images: [String]?
  - createdAt: Date?
  - resolvedAt: Date?
```

### `careers` (already exists on web, iOS should read)
```
careers/{careerId}
  - title: String
  - description: String
  - type: String? ("full-time" | "part-time" | "internship" | "volunteer")
  - isActive: Bool
  - createdAt: Date?
  - updatedAt: Date?
```

### `comments` (already have model, need to use)
```
comments/{commentId}
  - entity: String ("transaction" | "period" | "lead" | "building")
  - entityId: String
  - authorUserId: String
  - authorName: String? (denormalized)
  - text: String
  - createdAt: Date
```

---

## 6. Data Integrity Issues

### Inconsistent date formats
The web stores dates as Firestore Timestamps, ISO strings, epoch numbers, or `{seconds, nanoseconds}` maps. Our `FlexibleDate` decoder handles all cases, but:
- **Problem:** We can't reliably sort by date in Firestore queries — must sort client-side
- **Fix:** Web team should standardize on Firestore Timestamps

### Duplicate collections (prefixed vs unprefixed)
- `ady_transactions` vs `transactions` — we merge by ID, but this is fragile
- **Fix:** Web team should standardize on one naming convention. If `ady_` prefix is needed, both should use it

### `systemUsers.passwordHash` and `password`
- Legacy fields from pre-Firebase Auth era
- `password` is plaintext (used for migration only)
- **Fix:** After confirming all users have Firebase Auth accounts, remove these fields

### `Store` model is stale
- We removed Stores from Riad Shannak tabs but the model and collection still exist
- **Verdict:** Keep for ADY but don't show in Riad Shannak

---

## 7. Summary — Action Items

### Remove (dead code):
- `Service` model — no collection, no UI
- `AppTheme` model — no collection, no UI
- `ProjectFeature` model — no collection, no UI
- `ProjectPayment` model — no collection, no UI
- `ProjectDoc` model — no collection, no UI (use `PortalDocument`)
- `TeamMember` model — no collection, no UI

### Add fields to existing models:
- `SystemUser`: avatarUrl, lastLoginAt, loginMethod
- `Building`: floorsCount, elevator, parking, imageUrls, totalRentAmount
- `CRMLead`: estimatedCloseDate, tags, lastContactedAt
- `Transaction`: receiptNumber, paymentMethod, approvedBy, approvedAt
- `JobSubmission`: interviewDate, interviewNotes, rejectionReason, hiredAt
- `ContactInquiry`: repliedAt, repliedBy, priority

### Add new collections:
- `buildings/{id}/units` — unit management
- `tenants` — tenant management
- `rentPayments` — rent tracking
- `maintenanceTickets` — maintenance requests
- Read `careers` — job postings (already on web)
- Use `comments` — we have the model, just need UI

### Build missing UI:
- Documents tab (model exists, collection exists, no view)
- Careers tab (collection exists on web, need iOS view)
- Contact Messages tab (model exists, collection exists, no dedicated tab)
- Installments view (model exists, should show in Budget detail)
- Contracts view (model exists, should show in CRM or separate tab)
