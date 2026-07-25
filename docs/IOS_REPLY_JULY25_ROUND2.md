# iOS Reply — July 25, 2026 (Round 2)

Acknowledging web team updates and confirming implementation status.

---

## 1. Updates received and acknowledged

The following new documents were pulled and reviewed:

- `IOS_UPDATE_JULY25_PAYMENT_STATUS.md` — PayPal default, status tracking, complete CF reference, Firestore collections
- `IOS_UPDATE_JULY25_PORTAL_CHANGES.md` — Student Portal, Booking Preview tab, User Management improvements, PayPal primary, no auto emails, production URLs
- `IOS_PORTAL_ALIGNMENT_JULY25.md` (updated) — 6-category structure with admin action matrix, cross-cutting rules
- `WEB_IOS_INTEGRATION_ROADMAP_JULY25.md` (updated) — PayPal primary flow, `resumeBookingPayment`, `sendBrandedPasswordReset`, student portal routing

---

## 2. iOS implementation status

### Already completed ✅

| Item | Status |
|---|---|
| `findBookingsByEmail` with `JSONDecoder` + `FirestoreTimestamp` | ✅ Done |
| `verifyBookingCode` CF integrated | ✅ Done |
| `paymentMethod` with new gateway semantics (tap/stripe/paypal/manual/free) | ✅ Done |
| PayPal flow: `createPayPalOrder` + `SFSafariViewController` + custom URL scheme | ✅ Done |
| `syncBookingToCalendar` called after booking confirmation | ✅ Done |
| Admin portal restructured to 4 categories (Dashboard, People, Bookings & Services, Settings) | ✅ Done |
| `SlotGroup` and `SlotOverride` models added | ✅ Done |
| `AuthManager` pre-check with `findUserByEmail` | ✅ Done |
| Registration shows "Sign in" prompt if `systemUsers` exists | ✅ Done |
| Book Translation tab (OCR + Google Translate WebView) | ✅ Done |
| Stub views for all admin sub-sections (14 files) | ✅ Done |
| Legacy admin views commented out to reduce scatter | ✅ Done |

### In progress / next up 🔄

| Item | Priority |
|---|---|
| **Student Portal** — route `candidate`/`student`/`graduate` to restricted portal | P1 |
| **`resumeBookingPayment`** CF call for unpaid bookings in student portal | P1 |
| **`sendBrandedPasswordReset`** action in admin User Management | P2 |
| **6-category admin structure** — add Operations and Website categories (currently 4) | P2 |
| **Booking Preview** tab in admin (optional) | P3 |
| **No auto emails** — remove any auto-send logic from booking confirmation | P1 |
| **Production URLs only** — audit all URLs for `https://the-rs.com` | P1 |
| **`payment_config/booking.defaultMethod`** — read this to determine primary payment | P2 |
| **Funnel Manager** — implement with `job_submissions` status groups | P2 |
| **User 360** — implement unified profile by email across collections | P2 |

---

## 3. Category structure — confirmed

We see the updated `IOS_PORTAL_ALIGNMENT_JULY25.md` now has **6 categories**:

1. Dashboard
2. People
3. Bookings & Services
4. Operations
5. Website
6. Settings

iOS currently has 4 categories implemented. We will add **Operations** and **Website** in the next iteration.

**Question:** Should `tap-config` stay under Settings (as in our 4-category version) or move to Operations (as in the 6-category version)? We see it listed under Operations in the alignment doc but under Settings in the roadmap. Please confirm.

---

## 4. `sendBookingConfirmation` CF

We see in the updated CF reference that `sendBookingConfirmation` exists but is **disabled in the auto flow**. Confirmed — iOS will not auto-send confirmation emails. Admins can call it manually if needed.

**No action needed from web team on this.**

---

## 5. New Cloud Functions to integrate

From the updated reference, these CFs are new to iOS:

| CF | Purpose | iOS plan |
|---|---|---|
| `resumeBookingPayment` | Resume unpaid PayPal booking | Will add to student portal |
| `sendBrandedPasswordReset` | Branded password reset email | Will add to admin user management |
| `sendBookingStatus` | Send booking status update email | Will add to calendar/booking actions |
| `sendCVFeedback` | Send CV review feedback | Will add to funnel manager |
| `sendInvitationEmail` | Send call invitation | Will add to funnel manager |
| `sendSubmissionResponse` | Send custom response to applicant | Will add to funnel manager |
| `sendOfferEmail` | Send offer email | Will add to funnel manager |
| `sendPasswordSetupEmail` | Send password setup email | Will add to user management |
| `sendTestEmail` | Test SMTP | Will add to communication/settings |
| `sendContactReply` | Reply to contact form | Will add to website manager |
| `sendQuotation` | Send quotation email | Will add to proposal generator |
| `deleteUserData` | Delete all user data + Auth | Will add to user-360 and user management |
| `extractCVData` | AI-extract CV data | Will add to funnel manager |
| `sendBrandedPasswordReset` | Branded reset email | Will add to user management |

---

## 6. Questions for web team

### Q1: `tap-config` category placement
Should `tap-config` be under **Settings** or **Operations**? The alignment doc says Operations, the roadmap says Settings.

### Q2: `paypal_config` admin UI
Is there a web admin screen for managing PayPal credentials (`paypal_config` collection)? If so, which category/tab? iOS currently has `AdminTapConfigView` under Settings — should we add a `PayPalConfigView` too?

### Q3: Student Portal — Community tab
The Student Portal includes a "Community" tab. What Firestore collection backs this? Is it `portal_messages` filtered by audience, or a separate `community_posts` collection?

### Q4: `resumeBookingPayment` — booking already paid
When a booking is already paid, `resumeBookingPayment` returns a redirect URL instead of a PayPal URL. Should iOS just show the booking details in this case, or open the URL in Safari?

### Q5: Booking Preview tab
Is `booking-preview` a permanent addition to the admin portal, or a temporary QA tool? Should iOS invest in building it, or skip it?

### Q6: `communication_status` collection
Is this collection per-email or per-user? Does it have a document ID pattern (e.g., `communication_status/{email}`)?

---

## 7. What iOS needs from web team next

1. **Answer Q1–Q6 above**
2. **Student Portal wireframes or screen specs** — what exactly does each tab show? (We have the tab list but need layout/field details)
3. **Client Portal specs** — `client` role gets a Client Portal with Overview, Sessions, Documents, Messages, Settings. Need details on what data each tab shows.
4. **`booking-preview`** — confirm if needed on iOS

---

*Pushed by iOS team — July 25, 2026.*
