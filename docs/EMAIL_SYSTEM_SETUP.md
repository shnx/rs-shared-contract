# Web → iOS: Email System Setup (Resend + Firebase Cloud Functions)

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Email infrastructure for CV feedback, invitations, and future notifications

---

## What We've Built

### Email Provider: Resend
- **Domain verified:** `contact.the-rs.com` (Resend domain ID: `f19f7760-40f3-4c3f-b4c8-da9fcb5005e0`)
- **Sender address:** `info@contact.the-rs.com`
- **Reply-to:** `info@contact.the-rs.com`
- **SMTP:** `smtp.resend.com:465` (SSL), user = `resend`, pass = Resend API key

### Firebase Cloud Functions (deployed to `us-central1`)
Two callable functions for sending branded emails:

1. **`sendCVFeedback`** — sends CV feedback email to applicant
   - Params: `email`, `studentName`, `feedback`, `invitationId`
   - Updates `studentInvitations` doc with status `sent` + feedback
   - Email includes branded HTML template (RS theme colors, Arabic logo) with CTA button to registration link

2. **`sendInvitationEmail`** — sends invitation email to student
   - Params: `email`, `studentName`, `invitationId`
   - Updates `studentInvitations` doc with status `sent`
   - Email includes branded template with registration CTA

### Security
- Both functions require Firebase Auth (`context.auth` check)
- API key stored as Firebase Functions secret (`SMTP_PASS`), not in code
- Only authenticated admins can trigger email sends

---

## What the iOS Team Needs to Know

### Option A: Use the Cloud Functions directly
The iOS app can call the same Firebase Callable Functions:

```swift
// Swift example
let functions = Functions.functions(region: "us-central1")

// Send CV feedback
functions.httpsCallable("sendCVFeedback").call([
    "email": "student@example.com",
    "studentName": "Ahmed Ali",
    "feedback": "Your CV needs...",
    "invitationId": "inv_123"
]) { result, error in
    // handle result
}

// Send invitation
functions.httpsCallable("sendInvitationEmail").call([
    "email": "student@example.com",
    "studentName": "Ahmed Ali",
    "invitationId": "inv_123"
]) { result, error in
    // handle result
}
```

**This is the recommended approach** — the web side already handles the SMTP transport, branded templates, and Firestore updates.

### Option B: Use Resend SDK directly on iOS
If you prefer to send emails from the iOS side:
- Install Resend Swift SDK (or use REST API directly)
- Use the same API key (stored securely in Firebase Remote Config or your own secret manager)
- Sender: `info@contact.the-rs.com`
- SMTP: `smtp.resend.com:465`, user: `resend`, pass: your API key

**⚠️ Warning:** Never hardcode the API key in the iOS app binary. Use Firebase Remote Config or a backend proxy.

---

## Future Plans

1. **Password reset emails** — currently using Firebase Auth's native `sendPasswordResetEmail` (sent from Firebase's default domain). We may switch to Resend for branded password reset emails.
2. **Notification emails** — course enrollment, membership expiry, new content alerts
3. **Bulk emails** — newsletter, announcements (may need Resend's batch API)
4. **Email templates** — we'll create a shared template system so both platforms use consistent branding

---

## Firebase Functions Secret Setup

The API key is stored as a Firebase Functions secret:

```bash
firebase functions:secrets:set SMTP_PASS
# Paste your Resend API key (re_xxxxx) when prompted

firebase functions:secrets:set SMTP_USER
# Enter: resend

firebase deploy --only functions
```

The secret is automatically injected as `process.env.SMTP_PASS` in the Cloud Functions runtime.

---

## Questions for iOS Team

1. **Do you want to use the Cloud Functions** (Option A) or send emails directly from iOS (Option B)?
2. **Do you need any additional email types** beyond CV feedback and invitations?
3. **Should we create a shared email template repo** so both platforms use the same HTML templates?

---

*Reply by adding a new `.md` file to `shared/docs/` — we'll pull and read it.*
