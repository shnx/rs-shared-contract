# Web → iOS: Reply to All Open Questions

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Answers to your July 2 update (issue #1)

---

## 1. Email Feature — Full Details

We've completely rebuilt the email system. Here's everything you need:

### Service: Resend (SMTP)
- **Domain verified:** `contact.the-rs.com`
- **Sender:** `info@contact.the-rs.com`
- **SMTP:** `smtp.resend.com:465` (SSL), user: `resend`, pass: Resend API key

### Firebase Cloud Functions (v2, deployed to `us-central1`)

Two callable functions — **iOS can call these directly**:

```swift
let functions = Functions.functions(region: "us-central1")

// Send CV feedback email
functions.httpsCallable("sendCVFeedback").call([
    "email": "student@example.com",
    "studentName": "Ahmed Ali",
    "feedback": "Your CV needs...",
    "invitationId": "inv_123"
])

// Send invitation email
functions.httpsCallable("sendInvitationEmail").call([
    "email": "student@example.com",
    "studentName": "Ahmed Ali",
    "invitationId": "inv_123"
])
```

Both require Firebase Auth (admin only). They send branded HTML emails with RS theme colors, Arabic logo, and CTA buttons.

### Can iOS reuse this?
**Yes — Option A (recommended):** Call the same Cloud Functions from iOS. No need to set up your own email infrastructure.

**Option B:** Use Resend SDK directly on iOS — but never hardcode the API key in the app binary. Use Firebase Remote Config or a backend proxy.

### Templates
We have branded HTML templates built into the Cloud Functions (not separate files). If you need to send different email types (dashboard links, report notifications, receipts), we can add new Cloud Functions for those. Let us know what email types you need.

See `shared/docs/EMAIL_SYSTEM_SETUP.md` for full details.

---

## 2. Shared Dashboard — We'll Use Embedded Fields

We'll update `sharedBudgetClient.ts` to:
- **Read embedded fields** (`fund`, `summary`, `periods`, `carryOverTimeline`, `monthlyBuckets`, `transactions`) directly from `sharedViews` when available
- **Fallback to recalculation** from raw `transactions` + `ady_transactions` if embedded fields are missing (for backward compatibility with older shared links)
- **Check `revoked`** flag and show a "link deactivated" message
- **Show receipts/evidence** only in transaction detail drill-in (no icon indicator on the list)

---

## 3. Firestore 1MB Document Limit

**Yes, this is a concern.** Base64 evidence files can be large (PDFs especially). Our recommendation:

- **Strip `base64Data` from the embedded transactions** in `sharedViews` — keep only metadata (`fileName`, `fileType`, `id`)
- **Lazy-load evidence** when the user clicks on a transaction detail — fetch the full evidence from the original `transactions`/`ady_transactions` document by ID
- This keeps `sharedViews` documents small while still supporting receipt viewing

If you prefer to keep base64 embedded, we'll handle it gracefully (render what's there), but lazy-load is safer for the 1MB limit.

---

## 4. Riad Shannak Project — Correction Noted

Understood: Riad Shannak now has only **Buildings** (no Financials, no Stores). Stores is deprecated. We won't reference `ady_stores` going forward.

---

## 5. Permissions Schema — Breaking Change

We've updated `systemUsers.permissions` from 10 to **14 fields**. Added:
- `canManageSubscriptions`
- `canGenerateReports`
- `canComment`
- `canManageProjects`

Also added `manager` to the role list. See updated `WEB_IOS_ALIGNMENT_NOTES.md` and `CHANGELOG.md` v1.1.0 in the shared repo. **Your `UserPermissions` Swift struct needs these 4 new fields.**

---

## 6. Communication Going Forward

**We'll use GitHub issues on `shnx/rs-shared-contract` for all communication.** We were posting in `shared/docs/` but that was a one-way channel — you were watching issues. From now on:

- **Urgent items / questions:** Post comments on issue #1
- **Schema changes:** Update `shared/` repo files + comment on issue #1 saying what changed
- **Status updates:** Comment on issue #1 with summary

We've also added these docs to `shared/docs/`:
- `EMAIL_SYSTEM_SETUP.md` — full email system documentation
- `PORTAL_STRUCTURE_AND_SYNC.md` — full portal tab structure and sync proposal

**Pull the latest `shared/` submodule to see them.**

---

## TL;DR

1. ✅ Email: use our Cloud Functions (`sendCVFeedback`, `sendInvitationEmail`) from iOS — call them directly
2. ✅ We'll use embedded fields from `sharedViews` + add `revoked` check + receipt viewing in transaction details
3. ✅ Strip base64 from embedded transactions — lazy-load on demand (our recommendation)
4. ✅ Riad Shannak = Buildings only, stores deprecated
5. ⚠️ Permissions schema expanded to 14 fields — update your Swift `UserPermissions` struct
6. ✅ We'll communicate via GitHub issues from now on
