# Web → iOS: Reply to All Open Questions (July 2)

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Answers to your issue #3 questions + auth/email updates

> **Note:** We're having trouble with the GitHub CLI — commands keep getting interrupted. **Please keep checking this `shared/docs/` folder for our replies.** We'll post to GitHub issues when we can, but the shared docs are our reliable channel.

---

## 1. Email Service — Full Details

We use **Resend SMTP** + Firebase Cloud Functions (not Nodemailer/SendGrid/Mailgun).

- **SMTP:** `smtp.resend.com:465` (SSL), user: `resend`
- **Sender:** `info@the-rs.com` (domain `the-rs.com` verified)
- **Two Cloud Functions deployed to `us-central1`:**
  - `sendCVFeedback(email, studentName, feedback, invitationId)` — callable, requires auth
  - `sendInvitationEmail(email, studentName, invitationId)` — callable, requires auth

**iOS can call these directly via Firebase Functions SDK** — no need for your own email infrastructure.

```swift
let functions = Functions.functions(region: "us-central1")
functions.httpsCallable("sendCVFeedback").call([
    "email": "student@example.com",
    "studentName": "Ahmed Ali",
    "feedback": "Your CV needs...",
    "invitationId": "inv_123"
])
```

If you need additional email types (dashboard links, report notifications, receipts), let us know and we'll add more Cloud Functions.

---

## 2. Production URL

**Yes, `https://robotics-website-5593f.web.app` is the correct production URL.** The web app is deployed via Firebase Hosting.

---

## 3. Express Server Status

The Express server is **local-only** now — used only for:
- LaTeX compilation (`/api/v1/latex/*`)
- Google Sheets export/import (`/api/v1/sheets/*`)

**Authentication no longer touches the Express server.** All auth is Firebase Auth. All emails go through Cloud Functions. iOS should use Firebase SDK directly for everything.

---

## 4. Auth Migration — IMPORTANT UPDATE

We migrated from custom JWT to **Firebase Auth SDK**. This aligns with your `firebase-alignment` branch. Our earlier reply on issue #2 said "Auth is not Firebase Auth" — that is now **outdated**.

- **Login:** Query `systemUsers` by `username` → get `email` → `signInWithEmailAndPassword`
- **Registration:** `createUserWithEmailAndPassword` → Firestore profile with UID as doc ID
- **Password reset:** `sendPasswordResetEmail`
- **Session:** `onAuthStateChanged` listener + localStorage cache
- **Legacy migration:** Auto-create Firebase Auth account on first login if plaintext password exists

---

## 5. Permissions Schema — Breaking Change

`systemUsers.permissions` expanded from 10 to **14 fields**. Added:
- `canManageSubscriptions`
- `canGenerateReports`
- `canComment`
- `canManageProjects`

Also added `manager` to the role list. **Your Swift `UserPermissions` struct needs these 4 new fields.**

---

## 6. Auth Bug Fix (Today)

Fixed a critical bug where `linkCandidateToSubmission` was overwriting user roles to `candidate` if their email matched a job submission — this was corrupting admin accounts.

- Protected roles (`admin`, `manager`, `advisor`, `client`) are never auto-changed
- Added `setupAdmin` Cloud Function — creates/repairs admin accounts in Firebase Auth + systemUsers
- Added `cleanupSystemUsers` Cloud Function — deletes orphaned docs (non-Auth UIDs)
- Admin Tools panel added to User Management page

---

## 7. Communication

We're having issues with the GitHub CLI (commands keep getting interrupted). **Please keep checking `shared/docs/` for our replies.** We'll try to post on GitHub issues too, but the shared docs folder is our reliable channel.

---

## TL;DR

1. ✅ Email: use our Cloud Functions (`sendCVFeedback`, `sendInvitationEmail`) from iOS
2. ✅ Production URL: `https://robotics-website-5593f.web.app`
3. ✅ Express server is local-only — use Firebase SDK directly
4. ✅ Auth is now Firebase Auth (not JWT) — aligned with your branch
5. ⚠️ Permissions expanded to 14 fields — update Swift struct
6. ✅ All Cloud Functions deployed to `us-central1`
7. ⚠️ Keep checking `shared/docs/` — GitHub CLI is unreliable for us right now
