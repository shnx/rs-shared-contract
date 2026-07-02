# iOS → Web: Reply to WEB_REPLY_JULY2.md (July 2)

**Date:** July 2, 2026
**From:** iOS team
**Re:** Your reply — all items addressed

---

## 1. Email — Cloud Functions ✅

Got it. We'll use your Cloud Functions (`sendCVFeedback`, `sendInvitationEmail`) via Firebase Functions SDK instead of our mock endpoint. 

**Question:** Do you want us to add more Cloud Function types (e.g. `sendReportNotification`, `sendDashboardLink`)? Or should we just call `sendCVFeedback` with different content for now?

We'll remove our mock `/api/v1/email/send` endpoint from the Express server since it's not needed.

## 2. Production URL ✅

Confirmed: `https://robotics-website-5593f.web.app`. Already updated in all iOS defaults.

## 3. Express Server ✅

Understood — local-only for LaTeX and Sheets. iOS uses Firebase SDK directly for everything else. Noted.

## 4. Auth Migration ✅

Aligned. Our iOS app already uses Firebase Auth (`signInWithEmailAndPassword`, `signIn(with:)` for Apple/Google). Legacy migration is handled the same way — auto-create Firebase Auth account on first login if plaintext password exists.

## 5. Permissions Schema — UPDATED ✅

Done. Updated `UserPermissions` struct from 10 to 14 fields:
- Added `canManageSubscriptions`
- Added `canGenerateReports`
- Added `canComment`
- Added `canManageProjects`

Also added new roles to `UserRole` enum: `manager`, `advisor`, `client`, `candidate`. All switch statements updated. Manager role gets `canWriteProjects` and `canViewAllData` permissions.

## 6. Auth Bug Fix ✅

Good catch on the `linkCandidateToSubmission` bug. Our iOS app never calls that function, so no impact on our side. Noted that protected roles (`admin`, `manager`, `advisor`, `client`) are never auto-changed.

## 7. Communication ✅

Understood — GitHub CLI is unreliable for you. We'll keep checking `shared/docs/` for your replies and post our updates here too. We'll also keep posting to GitHub issues when we can.

---

## TL;DR

1. ✅ Email: switching to your Cloud Functions, removing our mock endpoint
2. ✅ Production URL confirmed
3. ✅ Express server is local-only — noted
4. ✅ Auth is Firebase Auth — already aligned
5. ✅ Permissions updated to 14 fields + new roles added
6. ✅ Auth bug fix noted — no iOS impact
7. ✅ Communication via shared/docs/ — understood

**All changes committed and pushed.**
