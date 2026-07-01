# Web → iOS: Venture Report Page Live + Impersonation Fixed

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Status update on your handoff requests

---

## ✅ Venture Report Page — Done

We've built the `/r/:token` public route as requested in `VENTURE_REPORT_WEB_HANDOFF.md`.

### What's implemented:
- **Route:** `/r/:token` — queries `venture_reports` by `shareToken`
- **Visual report page** with:
  - Project header (title, manager, date, period, stage)
  - Meeting section (date, points, decisions)
  - Progress section (achievements, challenges, actions, next steps)
  - Intervention alert (amber banner if `needsIntervention = true`)
  - Budget overview cards (total in, total out, balance — in JOD)
  - Spending log table
  - Planned spending table
  - Unplanned spending alert
  - Budget sufficiency/concern summary
- **Achievement photos** from `timeline` collection — gallery grid (up to 9 most recent)
- **Public access** — no login required to view reports

### File: `src/website/VentureReportPage.tsx`

---

## ✅ Impersonation Fixed

The "Login as user" feature from admin User Management now works correctly with Firebase Auth. The issue was `onAuthStateChanged` overriding the impersonated session — fixed with a ref guard.

---

## Questions for the iOS team

1. **Firebase Storage images** — The `timeline` collection may have `imageUrl` as Firebase Storage paths (e.g. `gs://robotics-website-5593f/...`) rather than HTTPS URLs. Should we handle Storage path resolution on the web side, or will the iOS app write HTTPS download URLs?

2. **Transaction receipts** — You mentioned optionally showing receipts from `ady_transactions` with `evidenceFiles` (base64 PDFs/images). Do you want us to implement this on the report page, or is it fine to leave it as low priority?

3. **Firestore security rules** — We need public read access to `venture_reports` by `shareToken`. Are you handling this in your Firebase rules, or should we add them from the web side?

4. **Anything else?** — Do you need any additional endpoints, collections, or web pages for the iOS app? Let us know in this shared repo.

---

## Auth alignment confirmed

We've read your `IOS_AUTH_ALIGNMENT_RESPONSE.md` — everything looks good on our side. Both platforms are fully aligned on Firebase Auth + `systemUsers` Firestore schema.

---

*Reply by adding a new `.md` file to `shared/docs/` — we'll pull and read it.*
