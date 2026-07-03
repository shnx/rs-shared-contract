# iOS → Web: Portal Improvement Ideas (July 3)

**From:** iOS team
**Re:** What we can improve — on web, on iOS, and together

---

## 1. ADY Project — Improvements

### On Web

#### A. Fix the 6 calculation bugs (still open)
We documented these in `IOS_CALCULATION_CODE_REFERENCE.md`. Quick recap:
1. **Carry-over between periods** — periods are cumulative, not isolated
2. **In Cash can be negative** — remove `max(0, ...)`, show red
3. **`payment` is income** — not expense
4. **Use date range for period assignment** — not just `periodId`
5. **Don't trust cached fields** (`spentAmount`, `revenue`, `netPosition`) — recompute from transactions
6. **Show unassigned transactions** — ones that don't fall in any period

#### B. Build the `/shared/{token}` page
Full spec in `SHARED_REPORT_PAGE_SPEC.md`. This is the #1 priority for web.

#### C. Achievements page needs transaction linkage
Your achievements page shows timeline entries but doesn't cross-reference with `ady_transactions`. It should:
- Show real amount from linked transaction (not from entry's `amount` field)
- Show "View Receipt" button when linked transaction has evidence
- Show "NOT IN BUDGET" badge for income/expense entries without `transaction_id`
- Show summary: Linked / With Receipt / Unlinked counts

#### D. Add a "Spending Forecast" widget on Dashboard
We have this on iOS — it uses historical completed periods to project future spend. The code is in `BudgetPeriodsView.swift` lines 594-630. It would be a great addition to the web dashboard.

#### E. Add cash flow timeline visualization
We compute a running balance within each period (income adds, expense subtracts, flag when it goes negative). This would look great as a line chart on the web. Code is in `BudgetPeriodsView.swift` lines 506-514.

#### F. Add the Venture Lab report export
We have a full PDF report builder on iOS. The web could generate the same report as a printable HTML page (which users can save as PDF via `window.print()`). The 7 sections are:
1. Project Information
2. Meeting with Sari Awwad
3. Achievements & Work Completed
4. Challenges & Risks
5. Support & Assistance Needed
6. Budget & Spending (with transaction log)
7. Budget Overview & Summary

#### G. Add meeting notes management
We have `ady_meetings` collection with `mainPoints` and `decisions` fields. The web should have a UI to create/edit meeting notes. These feed into the Venture Lab report.

### On iOS

#### A. Add charts/visualizations
The web has bar/line/pie charts. iOS only has cards and tables. We could add:
- Spending by category (pie chart)
- Income vs expense over time (bar chart)
- Cash flow running balance (line chart)
- Carry-over timeline (stacked bar)

SwiftUI has `Charts` framework (iOS 16+) which makes this straightforward.

#### B. Add bulk transaction import
Currently every transaction is entered manually. We could add:
- CSV import (parse and create transactions)
- Receipt scanning with OCR (extract amount, date, vendor from a photo)

#### C. Add transaction templates
For recurring expenses (rent, utilities, salaries), let the user save a template and quickly create a new transaction from it.

#### D. Add push notifications for budget alerts
- Notify when a period is 80% spent
- Notify when cash goes negative
- Notify when a period ends with debt

---

## 2. Achievements — Improvements

### On Both Platforms

#### A. Auto-create timeline entries from transactions
When a transaction is created with `fundingSource == "psut"`, automatically create a timeline entry with:
- `entry_type` = "income" or "expense" based on transaction type
- `transaction_id` = the new transaction's ID
- `title` = transaction description
- `amount` = transaction amount (fallback)
- `is_visible` = true

This would eliminate the "unlinked financial entry" problem.

#### B. Add image upload to achievements from iOS
Currently images are stored as URLs/Firebase Storage paths. iOS should be able to upload images directly when creating an achievement.

#### C. Add achievement categories filter
With 67 timeline entries, filtering by `entry_type` (milestone, income, expense, etc.) would help users find what they're looking for.

---

## 3. Riad Shannak — Improvements

### Both Platforms Need to Build This

The `buildings` collection exists but has 0 docs. We proposed a schema in `PORTAL_DATABASE_UI_PLAN.md`:

```
buildings/{id} → units subcollection → tenants, rent_payments, maintenance_tickets
```

**What we need to agree on:**
1. **Building schema** — name, address, units count, manager
2. **Unit schema** — number, floor, area, rent, tenant, lease dates
3. **Tenant schema** — name, phone, email, assigned unit
4. **Maintenance ticket schema** — building, unit, priority, status
5. **Rent payment schema** — unit, tenant, amount, date, method, status

**UI tabs for Riad Shannak:**
- Buildings (list + detail with units grid)
- Tenants (list + detail with rent history)
- Maintenance (kanban board: open → in-progress → resolved)
- Documents
- Achievements

**Who builds first?** We suggest web builds the CRUD UI first, iOS follows with a mobile-optimized version.

---

## 4. Tools & Website — Improvements

### On Web

#### A. Move localStorage data to Firestore
These tabs store data only in browser localStorage — they won't sync:
- Video Tutorials
- Video Assignments
- CV Template Manager
- Course Management
- Memberships
- CV Builder

**Proposed collections:**
| Data | Proposed Collection |
|---|---|
| Video Tutorials | `video_tutorials` |
| Video Assignments | `video_assignments` |
| CV Templates | `cv_templates` |
| Courses | `courses` |
| Lectures | `lectures` (subcollection of courses) |
| Memberships | `membership_plans` + `student_memberships` |
| CV Builder data | `student_cvs` |

#### B. Add reply function for contact submissions
You show contact submissions but can't reply. Add a reply form that sends an email.

#### C. Add CRM pipeline view
The `ady_crm` collection has leads with stages (new, interested, qualified, etc.). A kanban board view would be much better than a flat list.

### On iOS

#### A. Add video tutorial player
If video tutorials move to Firestore, iOS could show and play them.

#### B. Add CRM kanban on mobile
A swipeable kanban for CRM leads would be useful on mobile.

---

## 5. Shared Infrastructure — Improvements

### A. Move evidence files to Firebase Storage
Currently receipts/bills are stored as base64 in `ady_transactions.evidenceFiles[].base64Data`. This has a 1MB Firestore document limit problem.

**Proposal:**
- Upload evidence files to Firebase Storage at `evidence/{projectId}/{transactionId}/{fileId}`
- Store the download URL in `evidenceFiles[].downloadUrl` instead of `base64Data`
- Keep `fileName`, `fileType`, `fileSize` as metadata
- Both platforms read from the URL instead of decoding base64

**Migration plan:**
1. For each existing evidence file, upload to Storage and add `downloadUrl`
2. Keep `base64Data` for backward compatibility during migration
3. After both platforms are updated, remove `base64Data` from new docs

### B. Add real-time updates for shared report page
Currently the shared report is a snapshot (written once when the link is created). If we want it to update live:
- Option 1: The web page reads from `ady_transactions` directly (not from `sharedViews` snapshot)
- Option 2: iOS refreshes the `sharedViews` doc periodically
- Option 3: Use Firestore real-time listeners on the `sharedViews` doc

We prefer Option 1 for the shared page — read the source data directly. The `sharedViews` doc would just hold the token, `revoked` flag, and `reportTemplate`.

### C. Add audit logging
The web has `audit_log` (150 docs). iOS doesn't write to it. We should log:
- Transaction created/edited/deleted
- Period created/edited/deleted
- Share link created/revoked
- Meeting notes added/edited

### D. Add offline support on web
iOS works offline (Firestore offline cache). The web could use Firestore's offline persistence too:
```javascript
firebase.firestore().enablePersistence({ synchronizeTabs: true });
```

### E. Add dark mode
Both platforms should support dark mode. iOS has `@Environment(\.colorScheme)`. Web can use `prefers-color-scheme` CSS media query.

### F. Add role-based permissions for ADY
Currently anyone with access can see everything. We should add:
- **Admin:** full access (create/edit/delete everything)
- **Manager:** can view and edit transactions, cannot manage periods
- **Advisor:** read-only access, can view reports and shared links
- **Viewer:** can only view the dashboard

---

## 6. UX Improvements

### On Web

#### A. Add keyboard shortcuts
- `N` = new transaction
- `F` = focus search
- `E` = export current view
- `S` = share link

#### B. Add transaction search by amount range
"Show all transactions between JD 100 and JD 500" — useful for budget analysis.

#### C. Add period comparison view
Side-by-side comparison of two periods: "June vs July" showing income, expense, net, and category breakdown.

#### D. Add transaction split view
Sometimes one receipt covers multiple categories (e.g., JD 500 total = JD 300 equipment + JD 200 shipping). Allow splitting a transaction into multiple category lines.

### On iOS

#### A. Add swipe actions on transactions
- Swipe left: delete
- Swipe right: edit
- Swipe from right edge: view evidence

#### B. Add haptic feedback
- Success haptic when saving a transaction
- Warning haptic when cash goes negative
- Error haptic on failed operations

#### C. Add widget for iOS home screen
Show the fund card (PSUT grant progress) as a home screen widget.

#### D. Add Siri shortcuts
"Hey Siri, how much have we spent this period?"

---

## 7. Priority Ranking

Here's what we think the priorities should be:

### Immediate (This Week)
1. Web: Fix the 6 calculation bugs
2. Web: Build `/shared/{token}` page
3. Web: Add achievements ↔ transactions linkage

### Short Term (Next 2 Weeks)
4. Web: Add Venture Lab report as printable HTML
5. Web: Add meeting notes management
6. iOS: Add SwiftUI Charts to dashboard
7. Both: Move evidence files to Firebase Storage

### Medium Term (Next Month)
8. Both: Build Riad Shannak (buildings, units, tenants)
9. Web: Move localStorage data to Firestore
10. Both: Add role-based permissions
11. iOS: Add push notifications for budget alerts

### Long Term (Next Quarter)
12. iOS: Add receipt scanning with OCR
13. Both: Add transaction split view
14. Both: Add period comparison view
15. Web: Add CRM kanban board
16. Both: Add dark mode
17. iOS: Add home screen widget

---

## 8. What We Need From You

1. **Confirm which improvements you'll tackle** from the list above
2. **Tell us what you want us to build** that we don't have yet
3. **Agree on the Riad Shannak schema** so we can both start building
4. **Agree on moving evidence to Firebase Storage** — it's the biggest infrastructure improvement we can make
5. **Tell us if real-time shared report is needed** or if snapshot is fine for now

---

*Reply by adding a new `.md` file to `shared/docs/` — we'll pull and read it.*
