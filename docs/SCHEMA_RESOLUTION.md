# Schema Conflicts & Resolution (Aug 2, 2026)

This document addresses **known schema conflicts** between iOS and web implementations and establishes **canonical decisions** for each.

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

**Action**:
- [ ] Web: Audit code for `timeline_entries` references, update to `timeline`
- [ ] iOS: Remove `timeline_entries` fallback, use `timeline` only
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

**Action**:
- [ ] iOS: Migrate existing `ady_crm` docs to `crm_contacts` (cloud function or manual)
- [ ] iOS: Update code to write `crm_contacts` only, add optional `source` field
- [ ] Web: Confirm `crm_contacts` schema supports both workflows
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

**Action**:
- [ ] Update CHANGELOG to remove "conflict" — iOS recurrence model is canonical
- [ ] Update DATA_CONTRACT.md to document `availability` collection schema (recurring weekly)
- [ ] If web needs availability, use iOS schema as reference

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

**Status**: ⚠️ **Code-ready, but compilation will fail**

**Current state**:
- `LoginView.swift`: "Sign in with Google" button added
- `rsApp.swift`: OAuth URL handler fixed (uncommitted)
- `GoogleSignIn` SDK: **NOT in `.pbxproj`** — missing SPM dependency

**Resolution**: ✅ **Add GoogleSignIn SPM package to Xcode project**

**Action**:
- [ ] iOS dev (manual Xcode step): Add `https://github.com/google/GoogleSignIn-iOS` as SPM package
- [ ] iOS dev: Add URL scheme `com.googleusercontent.apps.531306594290-a6f79g782krg2cnkqnmf6oee3ju3pncp` to Info.plist
- [ ] Verify Firebase Console has Google sign-in provider enabled
- [ ] Commit rsApp.swift fix

---

## Summary Table

| Issue | Canonical Decision | Status |
|---|---|---|
| `timeline` vs `timeline_entries` | Use `timeline` everywhere | 🔴 Action needed |
| `ady_crm` vs `crm_contacts` | Use `crm_contacts` only | 🔴 Action needed |
| `transactions` + `ady_transactions` | Keep both, merge as-is | ✅ Working |
| Availability schema | Recurring weekly (dayOfWeek/startTime/endTime) | ✅ Canonical |
| Booking pricing (USD vs JOD) | Intentional split — valid | ✅ Canonical |
| Booking schema fields | Merge iOS + web schemas | 🔴 Action needed |
| Google Sign-In (iOS) | Add SPM package + URL scheme | 🔴 Action needed |

---

## Next Steps

1. **Update Firestore** if needed (CRM migration `ady_crm` → `crm_contacts`)
2. **Update iOS code** to use canonical collection names
3. **Update web code** to use canonical collection names
4. **Add missing SPM package** to iOS Xcode project (manual step)
5. **Merge PRs** (#9 booking funnel) with confirmed schemas
6. **Update CHANGELOG** to reflect resolutions
7. **Test both platforms** with synchronized code

---

*Document prepared: Aug 2, 2026*
*Next review: After schema alignment PRs merge*
