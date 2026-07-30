# Web Update — iOS Team — July 30, 2026

> **From:** Web Team
> **To:** iOS Team
> **Subject:** Login flow changes, PayPal capture error handling, email usage tracking, and booking management updates

---

## 1. Login Page — "Forgot your login?" merged flow

**Breaking change to login UI.** The separate "Forgot password" and "Forgot username" links have been merged into a single **"Forgot your login?"** flow.

### What changed

- Single input field accepts **email OR username**
- The `sendBrandedPasswordReset` cloud function now handles all recovery scenarios in one call
- No separate username lookup endpoint needed anymore

### iOS action

Replace any separate "Forgot Password" and "Forgot Username" buttons with a single **"Forgot your login?"** button that calls `sendBrandedPasswordReset` with the user's email or username.

---

## 2. `sendBrandedPasswordReset` — Enhanced with authProvider detection

**Cloud function updated.** The function now detects the user's `authProvider` and includes personalized instructions in the email.

### Scenarios handled

| User state | Email content |
|---|---|
| Firebase Auth user with password | Standard password reset link |
| Firestore profile, no Firebase Auth | "Set up your password" + login URL + username |
| Has `authProvider: 'google'` | Mentions "You signed in with Google — use the Google sign-in button" |
| Has `authProvider: 'apple'` | Mentions "You signed in with Apple — use the Apple sign-in button" |
| Only has CV submission, no account | "Create your account" + link to `/join-us` |
| No profile, no submission | Returns success silently (no user existence leak) |

### iOS action

If iOS has a password reset flow, it should call `sendBrandedPasswordReset` (not Firebase's built-in `sendPasswordResetEmail`). The function signature is unchanged:

```
callable: sendBrandedPasswordReset
payload: { email: string }
response: { success: boolean }
```

---

## 3. PayPal Capture — Comprehensive error handling

**Cloud function `capturePayPalOrder` updated.** Previously only handled `ORDER_ALREADY_CAPTURED`. Now returns structured error types with actionable messages for all common PayPal errors.

### New response shape

**Success:**
```json
{ "success": true, "captured": true }
```

**Error responses now include `errorType` and `message`:**

| `errorType` | PayPal error | What it means | Action |
|---|---|---|---|
| `NOT_APPROVED` | `ORDER_NOT_APPROVED` | Customer created order but never completed PayPal payment | Ask customer to re-book or send new payment link |
| `EXPIRED` | `ORDER_EXPIRED` | PayPal order expired (~3 hour limit) | Booking auto-cancelled. Create new booking. |
| `NOT_FOUND` | `RESOURCE_NOT_FOUND` | Order doesn't exist (sandbox/live mismatch or deleted) | Check PayPal config environment |
| `AUTH_ERROR` | HTTP 401 | PayPal credentials invalid | Check `paypal_config` in Firestore |
| `PENDING` | Capture status not COMPLETED | Payment held (e-check, held funds) | Check PayPal dashboard |
| `ORDER_ALREADY_CAPTURED` | `ORDER_ALREADY_CAPTURED` | Payment was already captured | Auto-marks booking as paid, returns success |
| Other | Various | Includes PayPal debug_id | Check PayPal debug ID in PayPal dashboard |

### iOS action

When calling `capturePayPalOrder`, check `result.success` first. If false, read `result.errorType` to show appropriate user-facing messages. Do not show raw error strings — use the `errorType` to display localized, actionable messages.

Example:
```swift
if result.success {
    // Booking confirmed
} else {
    switch result.errorType {
    case "NOT_APPROVED": showRebookPrompt()
    case "EXPIRED": showExpiredMessage()
    case "NOT_FOUND": showConfigError()
    case "AUTH_ERROR": showCredentialsError()
    case "PENDING": showPendingMessage()
    default: showGenericError(result.message)
    }
}
```

---

## 4. Email Usage Tracking — Communication Management

**New feature on web.** The Communication Management page now displays Resend free tier email usage:

- **Daily progress bar**: X / 100 emails sent today
- **Monthly progress bar**: X / 3,000 emails sent this month
- **Warning banner** when approaching limits (≤10 daily or ≤100 monthly remaining)
- **Today's sent emails list** with recipient, subject, type, and time

### Data source

Uses the existing `email_logs` Firestore collection. No new collections or cloud functions needed.

### iOS action

This is an admin-only feature. If iOS has a Communication Management admin tab, it can compute the same stats by querying `email_logs` where `status == 'sent'` and filtering by date. Resend free tier limits:
- **100 emails/day**
- **3,000 emails/month**

---

## 5. Booking cancellation — Already available

Confirmed that booking cancellation works across all admin views:
- **BookingsManagement** — Cancel button (XCircle icon) for any non-cancelled, non-completed booking
- **CalendarManagement** — Status change dropdown includes "cancelled", also deletes Google Calendar event
- **BookingCenter** (student portal) — Students can cancel their own bookings
- All cancellations send a cancellation email via `sendBookingStatusEmail`

### iOS action

If iOS doesn't yet have booking cancellation, implement it by:
1. Update booking document: `bookingDB.update(id, { status: 'cancelled' })`
2. Call `sendBookingStatusEmail` cloud function with status `'cancelled'`
3. If booking has a Google Calendar event (check `notes` for `Calendar: {eventId}`), delete it

---

## 6. User merging — PeopleManagement

The web admin has a **People Management** page with duplicate detection and merging:

- **Duplicate detection** by same phone, same name, or similar email
- **Merge action** reassigns all data from source email to primary email across 11 collections:
  `job_submissions`, `crm_contacts`, `student_crm`, `bookings`, `call_invitations`, `communication_status`, `user_messages`, `contact_submissions`, `candidate_posts`, `analytics_sessions`, `email_logs`
- Deletes the source `systemUsers` account and Firebase Auth account
- Updates `portal_messages` targetEmails arrays

### iOS action

This is admin-only. If iOS has user management, merging can be done by updating the email field across all listed collections. The merge is performed client-side using Firestore batch updates.

---

## Summary of iOS action items

| Priority | Item |
|---|---|
| P1 | Merge "Forgot Password" + "Forgot Username" into single "Forgot your login?" flow |
| P1 | Update `capturePayPalOrder` error handling to use `errorType` for user-facing messages |
| P2 | Add email usage tracking to Communication Management (if iOS has that tab) |
| P2 | Ensure booking cancellation calls `sendBookingStatusEmail` with `'cancelled'` |
| P3 | Consider adding user merge/duplicate detection to iOS admin |

---

Reply here with any questions.
