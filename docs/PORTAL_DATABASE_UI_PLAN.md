# iOS → Web: Database & UI Plan for the Portal (July 3)

**From:** iOS team
**Re:** What the database needs, what the UI should be on iOS and web, and how we keep building shareable things

---

## The Big Picture

We have one Firebase project (`robotics-website-5593f`), two apps (iOS + web), and a shared git repo for contracts. The portal serves multiple projects but **ADY is the most complex** — it has financial management, reporting, achievements, and now shareable links. This document is a plan for what the database needs to support and what the UI should look like on both platforms.

---

## Part 1: What the Database Needs

### 1.1 Core Collections (Already Exist)

These are working and both platforms read/write them:

| Collection | Purpose | Status |
|---|---|---|
| `ady_transactions` | Financial transactions (income/expense) | ✅ Working |
| `ady_periods` | Budget periods with date ranges | ✅ Working |
| `ady_documents` | Project documents (PDFs, images) | ✅ Working |
| `timeline_entries` | Achievements / project timeline | ✅ Working |
| `ady_meetings` | Meeting notes (with Sari Awwad) | ✅ Working |
| `sharedViews` | Shareable report link tokens | ✅ Working |
| `projects` | Project definitions | ✅ Working |
| `systemUsers` | User accounts, roles, permissions | ✅ Working |

### 1.2 What's Missing or Needs Fixing

#### A. `timeline_entries` — needs `transaction_id` populated

The `transaction_id` field exists in the model but many entries don't have it set. Income/expense timeline entries without a linked transaction are **"not in budget"** — they show up in achievements but don't count in financial totals.

**Action needed:**
- Web: When creating a timeline entry of type `income`/`expense`/`allocation`, also create a transaction and set `transaction_id`
- iOS: Same — when we add an achievement of financial type, we should create a linked transaction
- Migration: Audit existing entries, link to matching transactions by date+amount, or create missing transactions

#### B. `sharedViews` — needs `reportTemplate` field

We added `reportTemplate: "venture_lab"` to new share docs. The web should check this field to know which template to render.

**Future templates we plan to add:**
- `"dashboard"` — a live dashboard snapshot (charts, summaries)
- `"period"` — a single period report
- `"achievements"` — achievements-only page

For now only `"venture_lab"` exists. If the field is missing, default to `"venture_lab"`.

#### C. `ady_transactions` — evidence files are base64 inline

Receipts/bills are stored as base64 in `evidenceFiles[].base64Data`. This works but has limits:
- Large files bloat the document (Firestore 1MB doc limit)
- The shared report page needs to fetch these to show receipts

**Proposal:** For the shared report page, the web should:
1. Read `sharedViews/{token}` for the report data (includes `hasEvidence` and `evidenceFiles` metadata)
2. When user clicks a receipt button, fetch the full transaction from `ady_transactions/{id}` to get `base64Data`
3. Decode and display inline

**Future improvement:** Move evidence files to Firebase Storage and store download URLs instead of base64. This would require a migration but would solve the size limit. We're open to this if the web team agrees.

#### D. `ady_meetings` — needs `projectId` field

Currently meetings have `periodId` but not all meetings are tied to a specific period. We need `projectId` so meetings can be loaded at the project level (for the full report).

**Action:** We've updated the iOS app to load all meetings via `fetchAll()`. The web should do the same — query all meetings for the project, not just by period.

#### E. Collections we still need to design together

| Collection | Project | Status | Notes |
|---|---|---|---|
| `buildings/{id}/units` | Riad Shannak | Not built | Units within buildings |
| `tenants` | Riad Shannak | Not built | Tenant profiles |
| `maintenance_tickets` | Riad Shannak | Not built | Tenant-submitted tickets |
| `rent_payments` | Riad Shannak | Not built | Linked to units |
| `ady_summaries` | ADY | Not built | Period summary cache (optional) |

---

## Part 2: What the UI Should Be

### 2.1 ADY Project — Tab Structure (Both Platforms)

Both iOS and web should show the same tabs for ADY:

| Tab | Purpose | iOS Status | Web Status |
|---|---|---|---|
| **Dashboard** | Overview: fund card, period summary, recent transactions, quick stats | ✅ Done | ✅ Done |
| **Budget** | Period management: create/edit periods, allocated funds, carry-over timeline, planned items | ✅ Done | ✅ Done |
| **Transactions** | Full transaction explorer: add/edit/delete, filter, sort, group, evidence upload, period assignment | ✅ Done | ✅ Done |
| **Reports** | PDF/CSV exports, Venture Lab report, shareable link, per-period reports, monthly reports | ✅ Done | ⚠️ Needs shareable link page |
| **Achievements** | Visual timeline of milestones, linked to transactions, evidence access | ✅ Done | ⚠️ Needs transaction linkage |
| **Documents** | Project documents (PDFs, images) | ✅ Done | ✅ Done |

### 2.2 Dashboard Tab — UI Spec

```
┌─────────────────────────────────────────┐
│  PSUT Venture Lab Fund Card             │
│  [Progress bar: spent | cash | pending] │
│  Spent: JD 3,200  In Cash: JD 1,800    │
│  To Receive: JD 5,000                   │
│  (In Cash shows RED if negative)        │
├─────────────────────────────────────────┤
│  Period Summary                         │
│  ┌──────────┬────────┬────────┬───────┐ │
│  │ Period   │ Spent  │ Income │ Net   │ │
│  │ Jun 2026 │ JD 3200│ JD 5000│ +1800 │ │
│  └──────────┴────────┴────────┴───────┘ │
├─────────────────────────────────────────┤
│  Recent Transactions (last 5)           │
│  • 3 Jul  Cost  -JD 50.00  Cables       │
│  • 5 Jul  Payment  +JD 2000  Grant      │
│  • ...                                  │
├─────────────────────────────────────────┤
│  Quick Stats                            │
│  Total Income: JD 5,000                 │
│  Total Spent: JD 3,200                  │
│  Net Balance: JD 1,800                  │
└─────────────────────────────────────────┘
```

### 2.3 Budget Tab — UI Spec

```
┌─────────────────────────────────────────┐
│  Periods List                           │
│  ┌───────────────────────────────────┐  │
│  │ June 2026  [Active]               │  │
│  │ 1 Jun – 30 Jun                    │  │
│  │ Allocated: JD 5,000               │  │
│  │ Spent: JD 3,200 (64%)             │  │
│  │ [progress bar]                    │  │
│  │ Income: JD 5,000  Net: +JD 1,800  │  │
│  └───────────────────────────────────┘  │
│  (tap to see period detail)             │
├─────────────────────────────────────────┤
│  Carry-Over Timeline                    │
│  ┌──────────┬──────┬─────────┬────────┐ │
│  │ Period   │ Net  │ Carry In│CarryOut│ │
│  │ Jun 2026 │+1800 │ 0       │ +1800  │ │
│  │ Jul 2026 │-500  │ +1800   │ +1300  │ │
│  └──────────┴──────┴─────────┴────────┘ │
│  (red row if carryOut is negative)      │
├─────────────────────────────────────────┤
│  + New Period button                    │
└─────────────────────────────────────────┘
```

### 2.4 Transactions Tab — UI Spec

```
┌─────────────────────────────────────────┐
│  [Search bar]  [Filter: Type ▾] [Period▾]│
│  [Sort: Date ▾]  [Group: None ▾]        │
├─────────────────────────────────────────┤
│  Summary: 5 transactions                │
│  Income: +JD 5,000  Expense: -JD 3,200  │
│  Net: +JD 1,800                         │
├─────────────────────────────────────────┤
│  3 Jul  Cost   -JD 50.00   Cables       │
│         📎 Evidence  [PSUT]              │
│  5 Jul  Payment +JD 2,000  Grant        │
│         [PSUT]                           │
│  7 Jul  Bill   -JD 120.00  Electricity  │
│         📎 Evidence                      │
│  ...                                    │
├─────────────────────────────────────────┤
│  + Add Transaction                      │
└─────────────────────────────────────────┘
```

**Transaction detail (tap a row):**
- Full fields: type, amount, currency, date, category, vendor, funding source, note
- Evidence files (view/download)
- Period assignment
- Edit / Delete buttons

### 2.5 Reports Tab — UI Spec

```
┌─────────────────────────────────────────┐
│  SHAREABLE REPORT LINK                  │
│  ┌───────────────────────────────────┐  │
│  │ Public Report Page                │  │
│  │ [Create Link] [Manage]            │  │
│  │ https://...web.app/shared/{token} │  │
│  └───────────────────────────────────┘  │
├─────────────────────────────────────────┤
│  PSUT VENTURE LAB REPORT                │
│  ┌───────────────────────────────────┐  │
│  │ Project Progress & Budget Report  │  │
│  │ [Generate PDF] [Meeting with Sari]│  │
│  │ 2 meeting(s) will be included     │  │
│  └───────────────────────────────────┘  │
├─────────────────────────────────────────┤
│  PERIOD VENTURE LAB REPORTS             │
│  ┌───────────────────────────────────┐  │
│  │ June 2026  [Generate PDF]         │  │
│  │ July 2026  [Generate PDF]         │  │
│  └───────────────────────────────────┘  │
├─────────────────────────────────────────┤
│  PERIOD REPORTS                         │
│  ┌───────────────────────────────────┐  │
│  │ June 2026  [PDF] [CSV]            │  │
│  │ July 2026  [PDF] [CSV]            │  │
│  └───────────────────────────────────┘  │
├─────────────────────────────────────────┤
│  MONTHLY REPORTS                        │
│  ┌───────────────────────────────────┐  │
│  │ June 2026  expense/income/net     │  │
│  │ July 2026  expense/income/net     │  │
│  └───────────────────────────────────┘  │
├─────────────────────────────────────────┤
│  FULL PROJECT REPORT                    │
│  [PDF] [Google Sheets] [Monthly CSV]    │
└─────────────────────────────────────────┘
```

### 2.6 Achievements Tab — UI Spec

```
┌─────────────────────────────────────────┐
│  ADY Project Timeline                   │
│  12 entries                             │
├─────────────────────────────────────────┤
│  Linked: 8  |  With Receipt: 5  |  Unlinked: 2  │
├─────────────────────────────────────────┤
│  ● 15 Jun  💰 Income  +JD 2,000         │
│    ┃ PSUT Grant Installment             │
│    ┃ Second installment received        │
│    ┃ 🔗 Vendor: PSUT Venture Lab        │
│    ┃ 📎 View Receipt                    │
│  ● 10 Jun  🚩 Milestone                 │
│    ┃ Prototype v2 completed             │
│    ┃ First working prototype demo       │
│    ┃ [image]                            │
│  ● 5 Jun   💸 Expense  -JD 500          │
│    ┃ Equipment purchased                │
│    ┃ NOT IN BUDGET (red badge)          │
│  ● ...                                  │
└─────────────────────────────────────────┘
```

### 2.7 Documents Tab — UI Spec

```
┌─────────────────────────────────────────┐
│  [Search]  [Filter: Category ▾]         │
├─────────────────────────────────────────┤
│  📄 grant_agreement.pdf                 │
│     Category: Legal  |  2 Jul 2026      │
│  📄 budget_q2.pdf                       │
│     Category: Finance  |  1 Jul 2026    │
│  📷 team_photo.jpg                       │
│     Category: Media  |  28 Jun 2026     │
├─────────────────────────────────────────┤
│  + Upload Document                      │
└─────────────────────────────────────────┘
```

---

## Part 3: The Shareable Link — Our Vision

### What we built

The iOS app creates shareable links by writing a complete snapshot to `sharedViews/{token}`. The web renders it at `/shared/{token}`.

### What we want to keep building

We want to own the **creation** of shareable links from iOS. Here's our roadmap:

| Feature | Status | What web needs to do |
|---|---|---|
| Venture Lab report link | ✅ Done | Render `/shared/{token}` with the 7-section template |
| Clickable receipts in shared report | ✅ Data ready | Fetch `ady_transactions/{id}` for evidence when user clicks 📎 |
| PDF download from shared page | ⚠️ Web needs to implement | `window.print()` or `html2pdf.js` button |
| Period-specific shareable link | Planned | Same template but filtered to one period |
| Achievements-only shareable link | Planned | Render timeline entries with images |
| Dashboard snapshot shareable | Planned | Charts + summary cards |

### What we need from the web team NOW

1. **Build the `/shared/{token}` page** — full spec in `SHARED_REPORT_PAGE_SPEC.md`
2. **Check `revoked` on load** — show "link expired" if true
3. **Clickable receipt buttons** — fetch evidence from `ady_transactions` when clicked
4. **Download as PDF button** — top of the page
5. **Responsive design** — works on mobile and desktop

### What we'll handle on iOS

1. Creating/refreshing/revoking share links
2. Writing all report data to `sharedViews` doc
3. Adding new `reportTemplate` values as we build new shareable views
4. Updating the spec docs when we add new templates

---

## Part 4: Other Projects

### Riad Shannak (Real Estate)

**Current state:** `buildings` collection exists, read-only.

**What we need to design together:**

```
buildings/{buildingId}
  ├── name, address, unitsCount, occupiedUnitsCount
  ├── managerUserId
  └── units/ (subcollection)
        ├── unitNumber, floor, area, monthlyRent
        ├── tenantId, leaseStart, leaseEnd
        └── status (occupied/vacant/maintenance)

tenants/{tenantId}
  ├── name, phone, email
  ├── assignedUnitId, buildingId
  └── rentBalance

maintenance_tickets/{ticketId}
  ├── buildingId, unitId, tenantId
  ├── title, description, priority
  ├── status (open/in-progress/resolved)
  └── createdAt, resolvedAt

rent_payments/{paymentId}
  ├── unitId, tenantId, amount, date
  ├── method (cash/transfer/check)
  └── status (paid/pending/late)
```

**UI tabs for Riad Shannak:**
- Buildings (list + detail with units)
- Tenants (list + detail with rent history)
- Maintenance (ticket list + create)
- Documents
- Achievements

### Tools & Website

**Current tabs (both platforms):**
- Submissions (job applications)
- LaTeX Compiler
- Carousel (iOS-only for now)
- CRM (lead pipeline)
- Careers (job postings)
- Messages (contact form)

**No database changes needed here** — these collections are stable.

---

## Part 5: Sync Protocol

### How we keep in sync

1. **Shared git repo** (`rs-shared-contract`) — all schema docs, specs, and contracts live here
2. **When iOS adds a feature:** We write a spec doc → push to shared repo → web team pulls and implements
3. **When web adds a feature:** Same — write a doc → push → iOS team pulls
4. **Breaking changes:** Update `DATA_CONTRACT.md` first, both teams pull before deploying

### What we commit to

- Every new Firestore field or collection → documented in shared repo
- Every new shareable template → spec doc with full layout
- Every calculation change → update `ADY_CALCULATIONS_SPEC.md`
- Every UI tab change → update tab alignment docs

### What we ask the web team to commit to

- Check `sharedViews/{token}` for `reportTemplate` field and render accordingly
- Don't create shareable links from the web (iOS owns this)
- Use the exact calculation formulas from `ADY_CALCULATIONS_SPEC.md`
- Cross-reference `timeline_entries` with `ady_transactions` via `transaction_id`
- Flag unlinked financial entries (income/expense without `transaction_id`)

---

## Part 6: Open Questions for Discussion

1. **Evidence storage:** Should we migrate from base64 inline to Firebase Storage? Pros: no size limit, faster loads. Cons: migration effort, storage costs.

2. **Real-time updates:** Should the shared report page listen to Firestore changes (live) or just read once on load? iOS writes a snapshot — if we want live, we'd need to change the approach.

3. **Authentication for shared links:** Currently anyone with the URL can view. Should we add password protection or email gating? Or keep it fully public?

4. **Web creating timeline entries:** When the web creates a transaction, should it automatically create a linked timeline entry? Or should that be manual?

5. **Riad Shannak data model:** Can we agree on the building/units/tenants schema above so both teams can start building?

6. **Period summary cache:** Should we add an `ady_summaries` collection that caches period totals? Currently both platforms calculate on-the-fly. A cache would speed up the dashboard but needs invalidation logic.

---

## All Shared Docs (Reference)

| Document | What's in it |
|---|---|
| `DATA_CONTRACT.md` | Every Firestore collection, field, and merge strategy |
| `ADY_CALCULATIONS_SPEC.md` | Every formula, CSV format, common mistakes |
| `ADY_WEB_PORTAL_GUIDE.md` | Full overview of ADY data models and iOS views |
| `SHARED_REPORT_PAGE_SPEC.md` | Layout + implementation spec for `/shared/{token}` |
| `ACHIEVEMENTS_TRANSACTION_LINKAGE.md` | How achievements link to transactions |
| `IOS_TO_WEB_SUMMARY.md` | Summary of everything iOS built for web to implement |
| `PORTAL_STRUCTURE_AND_SYNC.md` | Web portal structure + sync proposal |
| `IOS_TAB_ALIGNMENT.md` | iOS tab structure + collection list |
| `WEB_IOS_ALIGNMENT_JULY3.md` | Web's alignment progress + questions for iOS |

---

*Reply by adding a new `.md` file to `shared/docs/` — we'll pull and read it.*
