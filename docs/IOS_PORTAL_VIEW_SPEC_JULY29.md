# iOS Portal View Specification — July 29

> **From:** Web Team
> **To:** iOS Team
> **Subject:** Complete portal view-by-view spec — what every screen in the iOS app must do to match or exceed the web portal

---

## Overview

The web portal (`StudentPortal`) has been significantly updated. The funnel manager has been redesigned with new stages, email-log-based welcome detection, and mobile-responsive layouts. This document specifies **every view** the iOS app must replicate, the exact data sources, Firestore collections, field names, and user actions for each.

**Golden rule:** If a user can do it on web, they must be able to do it on iOS — with the same naming, same data, same Firestore collections. No missing fields, no wrong function names, no skipped features.

---

## Authentication & Onboarding

### Login (`LoginPage.tsx` → iOS Login screen)

**Methods:**
1. **Email/Password** — Firebase Auth `signInWithEmailAndPassword`
2. **Google Sign-In** — popup on desktop, redirect on mobile. iOS should use native `ASAuthorizationGoogleIDProvider` or Firebase Auth `signInWithCredential` with Google ID token
3. **Apple Sign-In** — `ASAuthorizationAppleIDProvider` → Firebase Auth `signInWithCredential`

**Post-login routing:**
- New Google/Apple user → **Onboarding** (`/welcome` on web)
- Existing user → **Portal** (`/portal` on web)
- Admin/manager/advisor → **Admin Portal** (sidebar + tabs)
- All other roles (candidate, student, graduate, client) → **Student Portal** (no sidebar, no project selector)

**Critical:** `authProvider` must be set to `'google'` or `'apple'` for social sign-in users. These users skip the "Set Credentials" page and the password change section in Settings.

### Google Onboarding (`GoogleOnboardingPage.tsx` → iOS Onboarding flow)

**Steps (in order):**

1. **Username Step** (if `needsUsername === true`)
   - Input field for username (lowercase, min 3 chars, `[a-z0-9_]` only)
   - Debounced availability check (500ms) — query `systemUsers` collection where `username == value`
   - Pre-fill with email prefix (stripped to `[a-z0-9]`)
   - On confirm: update `systemUsers/{uid}` with `{ username, needsUsername: false }`
   - Errors: `too-short`, `invalid-chars`, `taken`

2. **Intro Step**
   - Welcome message with user's first name
   - Two option cards: "Upload your CV" and "Book a session"
   - "Get Started" button → CV step
   - "Skip to portal" button → navigate to portal

3. **CV Upload Step** (optional)
   - File picker (PDF, DOC, DOCX)
   - Explanation of why CV matters
   - On upload: convert to base64, create `job_submissions` doc with:
     ```
     { fullName, email, role: 'Google Onboarding', status: 'new',
       cvFileName, cvFileType, cvFileSize, cvBase64, submittedAt: new Date() }
     ```
   - Skip button → session step

4. **Session Booking Step** (optional)
   - Explanation of 30-min consultation
   - "Book a Session" button → navigate to booking page (`/book`)
   - Skip button → done step

5. **Done Step**
   - Success message
   - "Enter Portal" button → navigate to portal

**Bilingual:** All text must support Arabic (`ar`) and English (`en`). Use `isArabic` flag and `dir="rtl"` for Arabic.

---

## Student Portal — Tab-by-Tab Spec

The student portal (`StudentPortal.tsx`) has these tabs. Each tab is controlled by `candidateConfig` flags from `WebsiteConfigContext`. The iOS app must respect the same flags.

### Tab visibility config (Firestore: `site_config/candidate_config`)

| Flag | Tab | Default |
|------|-----|---------|
| `candidateShowSubmission` | Submission | true |
| `candidateShowSessions` | Sessions | true |
| (always) | Book Session | true |
| `candidateShowCV` | My CV | true |
| `candidateShowAskAdmin` | Ask Admin | true |
| `candidateShowCommunity` | Community | true |
| `candidateShowInbox` | Inbox | true |
| `candidateShowSettings` | Settings | true |

---

### 1. Submission Tab

**Data sources:**
- `submissionDB.getAll()` → filter by `email.toLowerCase() === currentUser.email`
- Sort by `submittedAt` descending, take first

**What to show:**
- **Status card:** Status badge with label and color (see status map below), role applied for, submission date, phone, job type, user's message
- **CV file card:** File name, file size (KB), Download button (opens base64 data URL)
- **Notes from team:** If `submission.notes` exists, show in amber box

**Status labels and colors (EXACT match):**
```
new → "Submitted" (blue)
reviewing → "Under Review" (amber)
shortlisted → "Shortlisted" (purple)
invited → "Invited" (green)
preparing → "Preparing" (indigo)
interested → "Interested" (teal)
declined → "Declined" (red)
archived → "Archived" (gray)
survey_completed → "Survey Completed" (cyan)
enrolled → "Enrolled" (emerald)
```

**If no submission:** Empty state with "Apply Now" button linking to `/join-us`.

**Actions:** Download CV (only action — read-only tab).

---

### 2. Sessions Tab

**Data sources:**
- `bookingDB.getByEmail(currentUser.email)` — sorted by `scheduledAt` descending

**What to show:**
- **"Book a New Session" CTA** at top — navigates to Book Session tab
- **Booking cards** for each booking:
  - Date (weekday, month, day)
  - Time + duration (e.g., "3:00 PM · 30 min")
  - Service type
  - Status badge: `confirmed` (green), `completed` (blue), `cancelled` (red), `pending` (amber)
  - Payment badge: `paid` (green) or `unpaid` (gray)
  - Price (if > 0)
  - Payment method (if not `free`)
  - **Join Session link** (if `status === 'confirmed'` and `sessionLink` exists)

**Actions per booking:**
- **If pending + unpaid:** "Pay with PayPal" button (calls `resumeBookingPayment(bookingId)` → redirect to PayPal URL) + "Confirm" button (navigates to `/book/confirm` with booking ID)
- **If confirmed or pending:** "Cancel" button (confirm dialog → `bookingDB.update(id, { status: 'cancelled' })` → send cancel email via `sendBookingStatusEmail`) + "Reschedule" button (opens `/reschedule?booking_id={id}`)

**If no bookings:** Empty state "No sessions yet" with prompt to book.

---

### 3. Book Session Tab (`PortalBookingPage.tsx`)

**This is a multi-step wizard. iOS must replicate ALL steps.**

**Data loaded on mount:**
- `bookingDB.getAll()` — for slot availability
- `firebaseDB.getById('payment_config', 'booking')` — default price
- `getBookingCalendarConfig()` — calendar config (available days, Berlin hours, slot interval, max date, block dates, limited slots mode)
- `serviceOfferingDB.getAll()` — active service offerings filtered by audience
- `slotOverrideDB.getAll()` — slot overrides (add/remove/block)
- `calendarBlockDB.getAll()` — recurring weekly blocks
- `listCalendarEventsViaCF(100)` — Google Calendar events for booked slots

**Steps:**

#### Step 1: Calendar
- Month calendar with navigation (prev/next month)
- Days are enabled/disabled based on:
  - `isDateBlocked`: past dates, after maxDate, within block range
  - `isDayAvailable`: available days config + slot overrides (add/remove)
  - Day availability level: `open` (green dot), `limited` (amber dot), `full` (no dot)
- On date select → show available time slots
- **Slot generation:** Berlin timezone hours (default 10:00–12:00), converted to user's local time. Interval from offering duration or config (default 30 min)
- **Slot availability:** Check against Google Calendar events, existing bookings, slot overrides (block type), and recurring calendar blocks
- **Slot limiting:** If `limitedSlotsMode` enabled, deterministically select N slots per day (seeded shuffle by date). If per-user `slotLimitEnabled`, select N per user per day (seeded shuffle by email+date)
- **Service offering selector:** Dropdown/segmented control to pick service type (if multiple visible)

#### Step 2: Form
- Name (pre-filled from `currentUser.name`)
- Email (pre-filled from `currentUser.email`)
- Notes (optional textarea)
- Coupon code field with "Apply" button
  - `couponDB` lookup by code
  - Types: `free` (price → 0), `fixed` (subtract value in cents), `percent` (percentage off)
  - Show effective price after discount

#### Step 3: Payment
- If free (coupon makes it 0): "Confirm Booking" button → creates booking directly
- If paid: "Pay with PayPal" button → `createPayPalOrder({ bookingId, email, name, scheduledAt, notes, finalPrice, lang })` → redirect to PayPal approval URL
- On PayPal return: `capturePayPalOrder` via cloud function
- **Booking creation:** `bookingDB.create({ name, email, serviceType, scheduledAt, durationMinutes, priceUSD, paymentStatus, status, notes, sessionLink, couponCode, userId })`
- **Email:** `sendBookingConfirmationEmail` on success
- **Calendar sync:** `syncBookingViaCloudFunction(bookingId)` to add to Google Calendar

#### Step 4: Success
- Show confirmation: name, email, date, local time label, calendar sync status
- "Done" button → return to Sessions tab

**Firestore collection:** `bookings`

---

### 4. My CV Tab (`MyCV.tsx`)

**Data sources:**
- `contact?.cvBase64` (from `crmContactDB` — CRMContact by email)
- `submission?.cvBase64` (from `submissionDB` — JobSubmission by email)
- Priority: contact CV > submission CV

**What to show if CV exists:**
- File name + Download button (base64 data URL)
- Replace button (file picker)
- **Preview:**
  - PDF: inline iframe/preview
  - Image: inline image display
  - Other: "Preview not available, click Download"
- **Team feedback:** If `contact.cvNotes` exists, show in amber box

**What to show if no CV:**
- Empty state with "Upload CV" button
- Link to full application form (`/join-us`)

**Upload action:**
- File picker (PDF, DOC, DOCX, PNG, JPG, JPEG)
- Max 5MB
- Convert to base64
- If contact exists: `crmContactDB.update(contact.id, { cvBase64, cvFileName })`
- If submission exists: `firebaseDB.update('job_submissions', submission.id, { cvBase64, cvFileName })`
- If neither: `crmContactDB.create({ name, email, role: 'Candidate', status: 'invited', accountCreated: true, cvBase64, cvFileName })`

**Firestore collections:** `crm_contacts`, `job_submissions`

---

### 5. Ask Admin Tab

**Data sources:**
- `userMessageDB.getByUser(currentUser.email)` — all messages by this user

**What to show:**
- **Compose form:**
  - Subject input (required)
  - Body textarea (required) — note: "links are auto-detected"
  - Attachment uploader (multiple files, max 2MB each, types: image/*, application/pdf, .doc, .docx, .txt)
  - Send button
- **Previous messages list:**
  - Subject + status badge (`sent` amber, `read` blue, `replied` green)
  - Body text (whitespace preserved)
  - **Auto-detected links:** URLs in body are rendered as clickable links. YouTube URLs get embedded video previews.
  - **Attachments:** Downloadable file chips with name and size
  - Timestamp
  - **Admin reply:** If `adminReply` exists, show in blue box with reply timestamp. Links in replies also auto-detected.

**Create message:**
```
userMessageDB.create({
  userId: currentUser.id,
  userEmail: currentUser.email,
  userName: currentUser.name,
  subject: subject.trim(),
  body: body.trim(),
  attachments: attachments.length > 0 ? attachments : undefined,
  status: 'sent',
  createdAt: new Date(),
})
```

**Firestore collection:** `user_messages`

**Attachment structure:**
```
{ id, fileName, fileType, fileSize, base64Data }
```

---

### 6. Community Tab

**Data sources:**
- `candidatePostDB.getAll()` — all community posts

**Categories (EXACT labels):**
- `question` → "Question" (blue)
- `discussion` → "Discussion" (purple)
- `sharing` → "Sharing" (teal)
- `advice` → "Advice" (amber)

**What to show:**
- Header with "New Post" button
- **New post form:**
  - Category selector (4 buttons)
  - Title input (required)
  - Body textarea (required) — links and YouTube URLs auto-detected
  - Attachment uploader (same spec as Ask Admin)
  - Post button + Cancel button
- **Post cards:**
  - Category badge + title
  - Body text
  - Auto-detected link previews (YouTube embeds + external links)
  - Attachments
  - Author avatar (first letter), author name, date
  - **Delete button** (only for post author)
  - **Replies:** Shown with left border, author info, body, link previews, attachments
  - **Reply form:** Inline input + attachment uploader + Reply button + Cancel

**Create post:**
```
candidatePostDB.create({
  authorId: currentUser.id,
  authorName: currentUser.name,
  authorEmail: currentUser.email,
  title: title.trim(),
  body: body.trim(),
  category,
  attachments: attachments.length > 0 ? attachments : undefined,
  links: extractLinks(postBody),
  replies: [],
  createdAt: new Date(),
})
```

**Add reply:**
```
candidatePostDB.addReply(postId, {
  authorId, authorName, authorEmail,
  body: replyBody.trim(),
  attachments: replyAttachments.length > 0 ? replyAttachments : undefined,
  links: extractLinks(replyBody),
  createdAt: new Date(),
})
```

**Delete post:** `candidatePostDB.delete(postId)` — only author can delete

**Firestore collection:** `candidate_posts`

---

### 7. Inbox Tab (`StudentInbox.tsx`)

**Data sources:**
- `portalMessageDB.getByAudience(currentUser.email, currentUser.role)` — messages targeted to this user

**What to show:**
- Header with unread count (red pulsing dot if unread > 0)
- **Filter tabs:** All / Unread / Read (with counts)
- **Message cards:**
  - Priority badge: `normal` (white/gray), `important` (amber), `urgent` (red)
  - Title + body preview
  - Timestamp
  - Read/unread indicator
- **Message detail view:** Full title, body, timestamp, priority badge
- **Mark as read:** On open, if `!readBy.includes(currentUser.email)` → `portalMessageDB.markAsRead(msg.id, currentUser.email)`

**Firestore collection:** `portal_messages`

**PortalMessage fields:** `id, title, body, priority, audience, targetEmails, targetRoles, readBy[], createdAt, expiresAt?`

---

### 8. Settings Tab (`StudentSettings`)

**What to show:**

- **Username field** — editable, lowercase only, `[a-z0-9_]`. Saved to `systemUsers/{uid}.username`.
- **Email field** — read-only. Shows different helper text:
  - Google/Apple users: "Your email is managed by Google and cannot be changed here."
  - Password users: "Email is tied to your login. Contact admin to change it."
- **Password section:**
  - Google/Apple users: Show "Signed in with Google" info box (no password fields)
  - Password users: New password + confirm password fields (min 6 chars). Uses `firebaseAuth.updatePassword(newPassword)`. Error `auth/requires-recent-login` → show "log out and log back in" message.
- **Additional actions (password users only):**
  - "Send Password Reset Email" button → `firebaseAuth.sendPasswordReset(email)`
  - "Link Google Sign-In" button → `firebaseAuth.linkWithGoogle()` → update `systemUsers/{uid}.authProvider = 'google'`
- **Save button** — saves username + password changes

**Firestore collection:** `systemUsers`

---

## Funnel Manager (Admin View) — New Stages

**For iOS admin users only.** The funnel manager has been completely redesigned.

### New Funnel Stages (replaces old stages)

| Stage | Label | Icon | Color | Meaning |
|-------|-------|------|-------|---------|
| `entered` | Entered | LogIn | blue | Submitted CV, booked a meeting, or signed up |
| `welcomed` | Welcomed | Mail | amber | Received ANY email from us (checked via email_logs) |
| `account_created` | Account Created | UserCheck | indigo | Has a portal account in systemUsers |
| `cv_uploaded` | CV Uploaded | FileText | purple | Has a CV on file |
| `session_booked` | Session Booked | Calendar | teal | Booked at least one session |
| `enrolled` | Enrolled | GraduationCap | emerald | Enrolled in a course or group of sessions |
| `declined` | Declined | XCircle | red | Opted out |

### Stage Computation (auto-computed, NOT manual)

```
function computeFunnelStage({ hasWelcome, hasAccount, hasCV, hasSession, isEnrolled, submission }) {
  if (submission?.status === 'enrolled' || isEnrolled) return 'enrolled';
  if (submission?.status === 'declined') return 'declined';
  if (hasSession) return 'session_booked';
  if (hasCV) return 'cv_uploaded';
  if (hasAccount) return 'account_created';
  if (hasWelcome) return 'welcomed';
  return 'entered';
}
```

### Welcome Detection (CRITICAL)

A person is "welcomed" if ANY of these are true:
1. `submission.welcomeSentAt` exists
2. `submission.invitedAt` exists
3. **ANY email in `email_logs` collection** where `to === person.email` AND `status === 'sent'`

This means all email templates count:
- Career Network Welcome
- Career Network Welcome + Free Call Coupon
- Talent Pool Welcome
- Welcome + Call Invite
- Account creation emails
- Any email sent from Communication Management

**iOS must replicate this logic.** Query `email_logs` collection, filter by `to` field (case-insensitive) and `status === 'sent'`.

### Data Loading (Unified)

The funnel loads from THREE sources, merged by email (lowercase, trimmed):

1. **Submissions** — `submissionDB.getAll()` (skip `status === 'archived'`)
2. **Bookings** — `bookingDB.getAll()` (match by `email` or `paypalPayerEmail`)
3. **System Users** — `firebaseDB.getAll('systemUsers')` (signups)

Each person is a `FunnelPerson`:
```
{
  email: string (lowercase),
  name: string,
  sources: ('submission' | 'booking' | 'signup')[],
  submission?: JobSubmission,
  bookings: Booking[],
  systemUser?: any,
  funnelStage: FunnelStage,
  enteredAt: Date,
  hasWelcome: boolean,
  hasAccount: boolean,
  hasCV: boolean,
  hasSession: boolean,
  isEnrolled: boolean,
}
```

### UI Layout

- **Desktop (md+):** Kanban board with columns per stage. Only shows columns that have people (activated stages).
- **Mobile:** Horizontal stage tabs + vertical card list. Only shows tabs with people.
- **Funnel summary bar:** Shows stage bubbles with counts and conversion percentages. Only shows stages that have people (Entered always shown).
- **Stage filter buttons:** Only shows buttons for stages that have people, plus "All".

### FunnelCard (each person)

Shows:
- Checkbox for selection
- Avatar (first letter of name)
- Name + email
- **Stage progress dots:** Row of icons (one per stage) with completed stages in green, current stage highlighted
- Stage label badge + source badges (submission/booking/signup)
- Entry date
- User 360 button (navigates to User360 view)
- Move-to menu (move to any other stage)
- Delete button (admin/manager only)

### Bulk Actions

- **Send Welcome:** Send welcome email to selected unreplied people. Template selector (4 templates). Updates `welcomeSentAt` on submission.
- **Create Accounts:** Create Firebase Auth accounts + systemUsers docs for selected people. Sends welcome email with credentials.
- **Export CSV:** Export selected people to CSV.
- **Delete:** Delete selected submissions (admin/manager only).

### Welcome Email Templates

| Template ID | Name | Subject |
|-------------|------|---------|
| `career_network` | Career Network Welcome | "Welcome to Our Career Network — Robotics Science" |
| `career_network_coupon` | Career Network Welcome + Free Call | "Welcome + Free Call Coupon — Robotics Science" |
| `talent_pool` | Talent Pool Welcome | "Welcome to Our Talent Pool — Robotics Science" |
| `call_invite` | Welcome + Call Invite | "We Liked Your CV — Free Call Invitation from Robotics Science" |

### Auto-Welcome Feature

- Toggle in header (stored in `localStorage` as `rs_autosend_welcome`)
- When enabled, auto-sends Career Network Welcome to unreplied submissions
- Unreplied = no `welcomeSentAt`, no `invitedAt`, status is `new`/`reviewing`/`shortlisted`/`preparing`

### Preset Filters

| Preset | Filter |
|--------|--------|
| Need Welcome | Stage = `entered` (no welcome sent) |
| Need Account | Stage = `welcomed` (welcomed but no account) |
| Need CV | Stage = `account_created` (has account but no CV) |
| Need Session | Stage = `cv_uploaded` (has CV but no session booked) |

### Advanced Filters

- **Has Account:** All / Yes / No
- **Has Bookings:** All / Yes / No
- **Source:** All / CV Submission / Booking / Sign-up
- **Date range:** From / To (entered date)

### Sort Options

- Newest first (date_desc)
- Oldest first (date_asc)
- Name A-Z (name_asc)
- Name Z-A (name_desc)

---

## Firestore Collections Reference

| Collection | Used By | Key Fields |
|------------|---------|------------|
| `job_submissions` | Submission tab, Funnel, MyCV | `email, fullName, status, cvBase64, cvFileName, welcomeSentAt, invitedAt, submittedAt, notes, role, phone, jobType, message` |
| `bookings` | Sessions tab, Book Session, Funnel | `email, name, serviceType, scheduledAt, durationMinutes, priceUSD, paymentStatus, status, sessionLink, paymentMethod, paypalPayerEmail, couponCode, userId, notes` |
| `systemUsers` | Auth, Settings, Funnel | `id (=uid), email, name, username, role, hasPassword, authProvider, needsUsername, projectIds, permissions, photoURL` |
| `crm_contacts` | MyCV, Funnel | `email, name, cvBase64, cvFileName, cvNotes, status, accountCreated` |
| `user_messages` | Ask Admin | `userId, userEmail, userName, subject, body, attachments, status, adminReply, repliedAt, createdAt` |
| `candidate_posts` | Community | `authorId, authorName, authorEmail, title, body, category, attachments, links, replies[], createdAt` |
| `portal_messages` | Inbox | `title, body, priority, audience, targetEmails, targetRoles, readBy[], createdAt, expiresAt` |
| `email_logs` | Funnel (welcome detection) | `to, subject, type, status, sentAt, submissionId, bookingId, contactId` |
| `coupons` | Book Session | `code, discountType, discountValue, isActive` |
| `service_offerings` | Book Session | `type, name, priceUSD, durationMinutes, audience, isActive` |
| `slot_overrides` | Book Session | `type (add/remove/block), date, endDate, startTime, endTime` |
| `calendar_blocks` | Book Session | `dayOfWeek, startTime, endTime` |
| `payment_config` | Book Session | `defaultPrice` |
| `site_config` | Config | `candidate_config (flags), admin_emails[]` |

---

## Critical Alignment Points

1. **Email matching is case-insensitive.** Always `.toLowerCase().trim()` before comparing emails.
2. **Welcome detection uses email_logs.** Do NOT only check `welcomeSentAt` — many users were welcomed via Communication Management or different templates.
3. **Funnel stages are auto-computed.** Do not store `funnelStage` in Firestore — compute it from the person's data each time.
4. **Only show activated stages.** Empty stages should not appear as columns/tabs in the UI.
5. **Booking slot times are in Berlin timezone.** Convert to user's local timezone for display. Store `scheduledAt` as ISO string.
6. **Attachments are base64-encoded.** Stored inline in Firestore. Max 2MB per file for messages, 5MB for CV.
7. **Link detection:** Any URL in message/post body should be auto-detected and rendered as a clickable link. YouTube URLs get embedded video previews.
8. **Bilingual support:** All user-facing text must support Arabic and English. Use `dir="rtl"` for Arabic text.
9. **Google/Apple users skip password fields.** `authProvider === 'google'` or `'apple'` → no password change UI, show "managed by Google" info instead.
10. **Admin roles:** `admin`, `manager`, `advisor` see the full admin portal with sidebar. All other roles see the student portal only.

---

## Questions or Clarifications

If anything is unclear, check the web source:
- `src/pages/StudentPortal.tsx` — main student portal
- `src/pages/PortalBookingPage.tsx` — booking wizard
- `src/pages/MyCV.tsx` — CV tab
- `src/pages/StudentInbox.tsx` — inbox
- `src/pages/FunnelManager.tsx` — funnel manager (admin)
- `src/website/GoogleOnboardingPage.tsx` — onboarding flow
- `src/contexts/AuthContext.tsx` — auth logic
- `src/utils/database.ts` — all DB methods
- `src/utils/emailTemplates.ts` — email template generators

Reply here with any questions.
