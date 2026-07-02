# Web → iOS: Shared Link Fixes + Firestore Rules + CV Feedback System

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Shareable link fixes, Firestore security rules, and new CV feedback feature

---

## ✅ Shareable Budget Dashboard — Fixed

### Issue
The `/shared/:token` route was querying Firestore directly from the client, but the `SHARED_LINK_GUIDE.md` recommended restricting `sharedViews` reads to server-only. This created a conflict — if strict rules were deployed, the shared link page would fail with permission errors.

### Fix
Updated `firestore.rules` to explicitly allow **public read** on three collections needed for shareable links:

```
match /sharedViews/{docId} {
  allow read: if true;  // Public — token-based access
  allow create: if request.auth != null;
  allow update, delete: if request.auth != null;
}

match /venture_reports/{docId} {
  allow read: if true;  // Public — shareToken-based access
  allow write: if request.auth != null;
}

match /timeline/{docId} {
  allow read: if true;  // Public — for achievement photos
  allow write: if request.auth != null;
}
```

All other collections now require `request.auth != null`.

### Firestore Indexes
Added `firestore.indexes.json` with indexes for:
- `sharedViews` by `token`
- `venture_reports` by `shareToken`
- `transactions` / `ady_transactions` by `projectId`
- `periods` / `ady_periods` by `projectId`
- `documents` / `ady_documents` by `projectId`
- `timeline` by `projectId` + `createdAt` (desc)

### What the iOS team needs to do
1. **Deploy the updated rules**: `firebase deploy --only firestore:rules,firestore:indexes`
2. **Continue creating `sharedViews` documents** with the same schema — the web side will read them
3. **No changes needed** to how you create share tokens — the web client reads `sharedViews` by `token` field

---

## ✅ CV Feedback & Generation System — New Feature

### Overview
We've built a complete CV feedback and generation system:

1. **Admin CV Center** (`cv-preparation` tab) — admin reviews applicant CVs, extracts data via AI, edits, picks a template, compiles LaTeX to PDF, writes feedback, and sends a branded email
2. **Branded Email** — sent via Firebase Cloud Function (`sendCVFeedback`) from `info@the-rs.com` using Resend SMTP
3. **Student CV Builder** (`cv-builder` tab) — student fills in info, picks a template, generates LaTeX, compiles to PDF, downloads
4. **Student Onboarding** — `/register?invite=X` shows feedback + links to CV Builder

### New collections used
- `studentInvitations` — already existed, now includes `feedback` field
- `preparedCVs` — already existed, stores admin-prepared and student-built CVs
- `cvTemplates` — already existed, LaTeX templates

### New Firebase Cloud Functions
- `sendCVFeedback(email, studentName, feedback, invitationId)` — callable, requires auth
- `sendInvitationEmail(email, studentName, invitationId)` — callable, requires auth

### What the iOS team needs to know
- The `studentInvitations` collection now has an optional `feedback` text field
- Cloud Functions are deployed to `us-central1`
- If the iOS app has a similar CV feature, it can use the same `cvTemplates` and `preparedCVs` collections

---

## ✅ User Management — Working

The User Management page is fully functional:
- Create/edit/delete users with Firebase Auth
- Bulk migration of passwordless users
- Impersonation (login as user)
- Candidate submissions tab
- Uses secondary Firebase Auth instance to avoid signing out admin

---

## Next steps for iOS team
1. Pull the latest `shared/` submodule
2. Deploy updated Firestore rules from the web repo
3. Let us know if you need any additional collections or endpoints

---

*Reply by adding a new `.md` file to `shared/docs/` — we'll pull and read it.*
