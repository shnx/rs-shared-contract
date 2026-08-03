# Schema Conflicts & Resolution (Aug 2, 2026)

This document addresses **known schema conflicts** between iOS and web implementations and establishes **canonical decisions** for each.

> **Code audit, Aug 3, 2026.** The resolutions below were written from the docs and
> CHANGELOGs, and several described a code state that no longer matches the
> repositories. Every item was re-checked against `RS_website@claude/tender-varahamihira-acb36d`
> and `RS-IOS-APP@devin-1`; the findings are recorded per item as **Audit (Aug 3)**.
> Net result: only item 6 (canonical `bookings` shape) is genuinely open in code —
> items 1 and 2 need a *data* migration, not a code change.

---

## Conflicts Found & Resolutions

### 1. **Timeline Collection Naming**

**Conflict**: Both `timeline` and `timeline_entries` exist in web code; iOS uses `timeline`; docs are unclear which is canonical.

**Current state**:
- Web: `VentureReportPage.tsx` reads `timeline` collection (0 docs)
- Web: `timelineSync.ts` syncs transaction ↔ timeline entries
- iOS: Reads from `timeline` or `timeline_entries` (tries both in fallback order)
- Real data: `timeline_entries` has 67 docs with achievements

**Resolution**: ✅ **Use `timeline` exclusively**
- All platforms use `timeline` collection name
- Web: Rename any internal references from `timeline_entries` to `timeline`
- iOS: Update to read `timeline` only (no fallback)
- Reason: v1.4.0 CHANGELOG declared the rename; now enforce it

**Audit (Aug 3)**: Both code-side actions are already satisfied — web has **zero**
`timeline_entries` references, and iOS Swift reads no timeline collection at all
(`ProjectsView.swift` only uses "Timeline" as a section title; there is no
fallback to remove). Web reads/writes `timeline` from `Timeline.tsx`,
`AccountingHub.tsx`, `Budget.tsx` and `sharedBudgetClient.ts`.
What remains is data, not code: the 67 achievement docs still live in
`timeline_entries`, which nothing now reads, so they are invisible in both apps.

**Action**:
- [x] Web: audit code for `timeline_entries` references — none exist
- [x] iOS: remove `timeline_entries` fallback — no timeline reads in Swift
- [ ] **Migrate the 67 `timeline_entries` docs into `timeline`**, then delete the old collection
- [ ] Update CHANGELOG with audit results

---

### 2. **CRM Collection Naming**

**Conflict**: iOS uses `ady_crm` (prefixed); Web uses `crm_contacts` (unprefixed); iOS maintains a split read from both.

**Current state**:
- Web: `crm_contacts` (submission workflow, web-written)
- iOS: `ady_crm` (manual leads, iOS-written) + reads `crm_contacts` (submission workflow)
- iOS: Deduplicates by email; prefers `crm` over `crm_contacts` when colliding

**Resolution**: ✅ **Use `crm_contacts` for all CRM data**
- Both platforms write to `crm_contacts` (unprefixed, web-canonical)
- iOS: Add `source` field ("manual" vs "submission") if needed to distinguish workflows
- iOS: Stop reading `ady_crm`; stop prefixing new CRM writes
- Reason: Follows convention (unprefixed = web-canonical), avoids split reads

**Audit (Aug 3)**: No Swift file references `ady_crm` — the only hits are in iOS
*markdown docs*. Web already reads and writes `crm_contacts` exclusively
(`database.ts:1585-1626`); its one leftover is the unused constant
`CRM_LEADS: 'ady_crm'` at `database.ts:62`. So the split read no longer exists in
code, and again only the data migration is outstanding.

**Action**:
- [ ] **Migrate any existing `ady_crm` docs to `crm_contacts`** (cloud function or manual)
- [x] iOS: code writes `crm_contacts` only
- [ ] Web: drop the dead `CRM_LEADS: 'ady_crm'` constant; add the optional `source` field if the workflows need distinguishing
- [ ] Update iOS docs that still describe `ady_crm` as the CRM collection
- [ ] Update CHANGELOG with migration approach

---

### 3. **Transactions Prefix Strategy**

**Status**: ✅ **No change needed** — already working as designed

**Current state** (from DATA_CONTRACT.md):
- Web writes to `transactions` (unprefixed)
- iOS writes to `ady_transactions` (prefixed)
- iOS reads **both** and merges by document ID, keeping newer `updatedAt`

**This is intentional and working**. Web queries hit `transactions`; iOS reads both and merges. No conflict.

---

### 4. **Availability Schema (Recurring Weekly vs Timestamps)**

**Conflict**: iOS docs claim iOS writes concrete time slots (`startDate`/`endDate` timestamps); CHANGELOG v1.5.0 lists as open question. Reality: iOS code uses recurring weekly.

**Current state**:
- iOS `AvailabilityRule` struct: `dayOfWeek` (0–6), `startTime`/`endTime` (strings "HH:mm"), `bufferMinutes`, etc.
- Web: Not found in code (likely not implemented yet)
- Google Calendar OAuth (iOS): Requires recurring weekly abstraction
- CHANGELOG v1.5.0: Lists as conflict — "iOS writes timestamps vs web uses recurring weekly"

**Resolution**: ✅ **Canonical: Recurring weekly (`dayOfWeek`/`startTime`/`endTime` as strings)**
- iOS implementation is correct; it matches Google Calendar and is more flexible
- Web should use same schema if implementing availability
- Reason: Google Calendar integration requires recurrence patterns; timestamps are inflexible

**Audit (Aug 3)**: "Web: Not found in code" is wrong — web implements availability
and already uses this exact schema: `AvailabilityManagement.tsx` edits
`dayOfWeek` / `startTime` / `endTime` / `bufferMinutes` / `maxBookingsPerDay` /
`priceTier`, alongside `SlotOverridesManagement.tsx`, `CalendarBlocksManagement.tsx`,
`TimeSlotConfigManagement.tsx` and `PortalBookingPage.tsx`. The platforms are
already aligned; this is a confirmation, not a target.

**Action**:
- [ ] Update CHANGELOG to remove "conflict" — iOS recurrence model is canonical
- [ ] Update DATA_CONTRACT.md to document `availability` collection schema (recurring weekly), citing the web implementation as the reference
- [x] Web availability exists and matches the canonical schema

---

### 5. **Booking Pricing (USD vs JOD Split)**

**Conflict**: `bookings` collection uses USD pricing; `time_slot_bookings` uses JOD; no unified model.

**Current state**:
- `bookings` (consultations): `priceUSD` (web PR #9)
- `time_slot_bookings` (student slots): `originalPriceJOD`, `finalPriceJOD` (v1.4.0)
- No unified currency handling

**Resolution**: ✅ **Two separate use cases, both valid**
- **`bookings` (public consultations)**: USD pricing (external clients, international)
- **`time_slot_bookings` (student slots)**: JOD pricing (internal educational program)
- No merge needed; these are intentionally separate features
- Conversion: If comparing across them, use 0.707 USD→JOD rate (documented in DATA_CONTRACT.md)

**Action**:
- [ ] Document this as intentional separation in CHANGELOG
- [ ] Ensure both PRs (#9 booking and v1.4.0 slots) are merged successfully
- [ ] No code changes needed — design is correct

---

### 6. **Bookings Schema Alignment**

**Conflict**: iOS `BookingFull` model has fields (`consultationId`, `tier`, `priceEUR`, `paymentMethod`, `invoiceUrl`) not in web `bookings` schema.

**Current state**:
- iOS v1.6.0 CHANGELOG: Lists extra fields as "needs alignment"
- Web PR #9: Adds `bookings` collection and tab, but schema undefined
- No clear canonical shape

**Resolution**: ✅ **Merge iOS and web schemas**

**Canonical `bookings` schema**:
```jsonc
{
  "userId":          "string?",           // Firebase Auth UID (null = guest)
  "email":           "string",            // Always present for linking
  "name":            "string",
  "company":         "string?",
  "serviceOfferingId": "string",          // Links to service_offerings
  "serviceType":     "string",            // Denormalized from offering
  "scheduledAt":     Timestamp,           // When the consultation/booking is scheduled
  "durationMinutes": number?",
  "status":          "pending|confirmed|completed|cancelled", // Booking status
  "priceUSD":        number?",            // Original price in USD (public consultations)
  "finalPrice":      number?",            // After discounts/coupons (in whatever currency)
  "couponCode":      "string?",
  "paymentStatus":   "unpaid|paid|refunded",
  "paymentMethod":   "tap|stripe|paypal|manual|free", // Provider
  "tapChargeId":     "string?",
  "tapUrl":          "string?",
  "stripeSessionId": "string?",
  "paypalOrderId":   "string?",
  "paypalTransactionId": "string?",
  "paypalPayerEmail": "string?",
  "sessionLink":     "string?",           // Zoom/Teams/etc. join link
  "notes":           "string?",
  "createdAt":       Timestamp?,
  "updatedAt":       Timestamp?
}
```

**Action**:
- [ ] Web PR #9: Use this schema for `bookings` collection
- [ ] iOS: Confirm `BookingFull` model aligns (add missing fields if needed)
- [ ] Both platforms use same field names and types

---

### 7. **Google Sign-In on iOS**

**Status**: ✅ **Done** — RS-IOS-APP `792543e` (Aug 3)

**Current state**:
- `LoginView.swift`: "Sign in with Google" button added
- `rsApp.swift`: OAuth URL handler fixed (uncommitted)
- `GoogleSignIn` SDK: **NOT in `.pbxproj`** — missing SPM dependency

**Resolution**: ✅ **Add GoogleSignIn SPM package to Xcode project**

**Audit (Aug 3)**: All code actions landed in `792543e`, which also fixed a latent
`accessToken.expirationDate` optional-comparison error in
`GoogleCalendarService.swift` and re-declared `CFBundleLocalizations` (en/ar),
which stopped reaching the bundle once `INFOPLIST_FILE` was set. Debug and Release
both build.

**Action**:
- [x] iOS: `GoogleSignIn-IOS` 7.1.0 SPM package added (`GoogleSignIn` + `GoogleSignInSwift`)
- [x] iOS: URL scheme `com.googleusercontent.apps.531306594290-a6f79g782krg2cnkqnmf6oee3ju3pncp` added via a real `Info.plist`
- [x] iOS: `rsApp.swift` OAuth URL handler committed
- [ ] Verify Firebase Console has Google sign-in provider enabled (console-side, not verifiable from code)

---

## Summary Table

| Issue | Canonical Decision | Status (audited Aug 3) |
|---|---|---|
| `timeline` vs `timeline_entries` | Use `timeline` everywhere | ✅ Code aligned · 🔴 67 docs still to migrate |
| `ady_crm` vs `crm_contacts` | Use `crm_contacts` only | ✅ Code aligned · 🔴 data migration + dead constant |
| `transactions` + `ady_transactions` | Keep both, merge as-is | ✅ Working |
| Availability schema | Recurring weekly (dayOfWeek/startTime/endTime) | ✅ Both platforms already match |
| Booking pricing (USD vs JOD) | Intentional split — valid | ✅ Canonical |
| Booking schema fields | Merge iOS + web schemas | 🔴 Only genuinely open code item |
| Google Sign-In (iOS) | Add SPM package + URL scheme | ✅ Done — `792543e` |

---

## Next Steps (revised after the Aug 3 audit)

1. **Migrate Firestore data** — `timeline_entries` → `timeline` (67 docs, currently
   unreachable from either app), and any `ady_crm` → `crm_contacts`. Both are data
   migrations; the code on both platforms is already on the canonical names.
2. **Adopt the canonical `bookings` shape** (item 6) — the one open code item.
3. **Document the `availability` schema** in DATA_CONTRACT.md, using the existing web
   implementation as the reference.
4. **Tidy stale references** — the dead `CRM_LEADS: 'ady_crm'` constant in web
   `database.ts`, and the iOS markdown docs that still name `ady_crm`.
5. **Update CHANGELOG** to reflect resolutions and this audit.
6. **Land the long-lived branches** — this contract describes behaviour that only
   exists on `RS_website@claude/tender-varahamihira-acb36d` (255 commits ahead of
   `main`) and `RS-IOS-APP@devin-1` (137 ahead). Until those merge, `main` on both
   apps does not implement anything here.

---

*Document prepared: Aug 2, 2026*
*Code-audited: Aug 3, 2026*
*Next review: After the data migrations and the `bookings` schema change land*
