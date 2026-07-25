# iOS Update — July 25, 2026: Payment, Status Tracking & New Features

Everything the iOS team needs to know about recent changes and how to track student/submission/booking status.

---

## 1. PayPal is now the default payment method

### What changed

- **PayPal is the default** for all public booking flows (Discover page, Portal booking, Meeting invitations).
- Tap and Stripe Cloud Functions still exist but are **not wired to the public booking flow**.
- The admin Payments tab (`tap-config` page) now has a **default payment method dropdown** (PayPal/Tap) stored in `payment_config/booking.defaultMethod`.
- The admin test payment button now creates a **PayPal test order** instead of a Tap test payment.

### PayPal credentials

PayPal credentials are stored in Firestore collection `paypal_config`:
```
paypal_config/{autoId}
  clientId: string
  clientSecret: string
  environment: "live" | "sandbox"
  isActive: boolean
  createdAt: timestamp
```

The `createPayPalOrder` Cloud Function reads from this collection at runtime. No Firebase secrets are used for PayPal.

### iOS PayPal flow

```
1. User selects slot + applies coupon
2. iOS calls createPayPalOrder({ bookingDetails })
3. CF creates pending booking in Firestore (or updates existing one)
4. CF creates PayPal order via PayPal v2 API
5. CF returns { success, url, orderId }
6. iOS opens url in SFSafariViewController / ASWebAuthenticationSession
7. User pays on PayPal
8. PayPal redirects back to the website (SITE_URL)
9. iOS polls bookings/{id}.paymentStatus until "paid"
```

### createPayPalOrder payload

```json
{
  "bookingDetails": {
    "email": "student@example.com",
    "name": "Student Name",
    "scheduledAt": "2026-08-01T10:00:00.000Z",
    "notes": "Consultation",
    "finalPrice": 4.50,
    "bookingId": "existing-booking-id"  // optional — if booking already created
  }
}
```

**Response:**
```json
{
  "success": true,
  "url": "https://www.paypal.com/checkoutnow?token=...",
  "orderId": "PAYPAL-ORDER-ID"
}
```

### Important: `finalPrice` is the coupon-discounted price

The iOS app must calculate the discounted price before calling `createPayPalOrder`. The CF uses `finalPrice` if provided, otherwise falls back to `payment_config/booking.defaultPrice`.

---

## 2. How to check the status of each student / submission / booking

### A. Job Submissions (candidates who submitted CVs)

**Collection:** `job_submissions`

**Status field:** `status` — one of:
| Status | Meaning |
|---|---|
| `new` | Just submitted, not yet reviewed |
| `reviewing` | Admin is reviewing the CV |
| `shortlisted` | Passed initial review |
| `preparing` | Being prepared for outreach |
| `invited` | Sent an invitation/welcome email |
| `interested` | Responded positively |
| `survey_completed` | Completed a survey/quiz |
| `enrolled` | Enrolled in a course/program |
| `declined` | Declined the offer |
| `archived` | No longer active |

**How to query:**
```swift
db.collection("job_submissions")
  .whereField("status", isEqualTo: "new")
  .getDocuments()
```

**Additional fields for tracking:**
- `reviewedAt` — when admin reviewed
- `reviewedBy` — which admin reviewed
- `notes` — admin notes
- `welcomeSentAt` — when welcome email was sent
- `extractedText` — AI-extracted CV text

### B. Student CRM (enrolled/active students)

**Collection:** `student_crm`

**Status field:** `status` — one of:
| Status | Meaning |
|---|---|
| `invited` | Invited to join |
| `active` | Currently active |
| `enrolled` | Enrolled in courses |
| `alumni` | Completed program |
| `inactive` | No longer active |

**Call tracking fields:**
- `callStatus`: `sent` | `scheduled` | `completed` | `expired`
- `callInvitationId` — links to `call_invitations` collection
- `callType`: `free` | `discounted`
- `callDiscountPercent`: number

**Course tracking:**
- `proposedCourses`: string[] — courses suggested to the student
- `enrolledCourses`: string[] — courses they're enrolled in
- `mentorshipUnlocked`: boolean
- `mentorshipUnlockedAt`: timestamp

### C. Bookings (consultation calls, meetings)

**Collection:** `bookings`

**Status field:** `status` — one of:
| Status | Meaning |
|---|---|
| `pending` | Created but not yet confirmed (awaiting payment) |
| `confirmed` | Payment confirmed / booking active |
| `completed` | Session has taken place |
| `cancelled` | Cancelled by admin or student |

**Payment status field:** `paymentStatus`:
| Status | Meaning |
|---|---|
| `unpaid` | Payment not yet received |
| `paid` | Payment completed |

**Payment method field:** `paymentMethod`:
`"tap"` | `"stripe"` | `"paypal"` | `"manual"` | `"free"`

**PayPal-specific fields:**
- `paypalOrderId` — PayPal order ID from createPayPalOrder
- `paypalTransactionId` — PayPal capture/transaction ID

**How to check if a student paid:**
```swift
db.collection("bookings")
  .whereField("email", isEqualTo: "student@example.com")
  .whereField("paymentStatus", isEqualTo: "paid")
  .getDocuments()
```

**How to poll for payment completion after PayPal redirect:**
```swift
db.collection("bookings").document(bookingId)
  .addSnapshotListener { snapshot, error in
    guard let data = snapshot?.data() else { return }
    if data["paymentStatus"] as? String == "paid" {
      // Payment confirmed — proceed
    }
  }
```

### D. Funnel stages (pipeline tracking)

The Funnel Manager on web groups submissions into stages. iOS should mirror this:

**Stage groups:**
| Group | Stages included |
|---|---|
| New Applications | `new`, `reviewing` |
| Shortlisted | `shortlisted`, `preparing` |
| Responded | `invited`, `interested` |
| Survey Done | `survey_completed` |
| Enrolled | `enrolled` |
| Inactive | `declined`, `archived` |

**How to list all candidates in a stage group:**
```swift
let stages = ["new", "reviewing"]
db.collection("job_submissions")
  .whereField("status", in: stages)
  .getDocuments()
```

### E. Communication / Email Funnel status

**Collection:** `communication_status`

**Fields:**
- `email`: string
- `funnelStatus`: `"hot"` | `"follow-up"` | `"cool"` | `"ok"`
- `notes`: string
- `nextAction`: string
- `updatedAt`: timestamp

This is a manual override the admin sets per person. iOS can read this to show the admin's assessment of where a person is in the communication pipeline.

### F. Meeting Invitations

**Collection:** `meeting_invitations`

**Status field:** `status`:
| Status | Meaning |
|---|---|
| `active` | Invitation sent, awaiting booking |
| `booked` | Recipient has booked a slot |
| `expired` | Invitation expired |
| `cancelled` | Cancelled by admin |

**Payment fields:**
- `requiresPayment`: boolean
- `priceUSD`: number

### G. Call Invitations

**Collection:** `call_invitations`

**Status field:** `status`:
| Status | Meaning |
|---|---|
| `sent` | Invitation sent |
| `scheduled` | Call scheduled |
| `completed` | Call completed |
| `expired` | Expired |

### H. Contact submissions (website inquiries)

**Collection:** `contact_submissions`

**Status field:** `status`:
| Status | Meaning |
|---|---|
| `new` | New inquiry |
| `replied` | Admin replied |
| `archived` | Archived |

### I. User Messages (student → admin)

**Collection:** `user_messages`

**Status field:** `status`:
| Status | Meaning |
|---|---|
| `sent` | Message sent, not yet read |
| `read` | Admin has read |
| `replied` | Admin has replied |

### J. Enrollment tracking

**Collection:** `enrollments`

**Status field:** `status`:
| Status | Meaning |
|---|---|
| `active` | Currently enrolled |
| `completed` | Course completed |
| `cancelled` | Enrollment cancelled |

**Fields:**
- `studentId`, `studentName`, `studentEmail`
- `progress`: number (0-100)

### K. Subscription plans

**Collection:** `subscriptions`

**Status field:** `status`:
| Status | Meaning |
|---|---|
| `trial` | On trial |
| `active` | Active subscription |
| `expired` | Expired |
| `cancelled` | Cancelled |

---

## 3. User 360 — unified student view

The `user-360` screen aggregates data across all collections by email. iOS should:

1. Search by email across `systemUsers`, `job_submissions`, `student_crm`, `bookings`, `call_invitations`, `portal_messages`, `email_logs`, `analytics_sessions`.
2. Show a unified profile with:
   - Account info (name, email, role, auth provider)
   - Submission status + funnel stage
   - Student CRM status (enrolled courses, call status)
   - All bookings (status, payment status, payment method)
   - All call invitations
   - Portal messages
   - Email history
   - Analytics sessions
3. Actions: send email, delete all data (`deleteUserData` CF).

---

## 4. Complete Cloud Function reference (updated)

| Use | CF Name | Auth | Notes |
|---|---|---|---|
| Public email search | `findBookingsByEmail` | None | Returns bookings by email |
| Send verify code | `sendBookingVerifyCode` | None | 6-digit code to email |
| Verify code | `verifyBookingCode` | None | Returns { verified: boolean } |
| PayPal pay | `createPayPalOrder` | None | Returns { success, url, orderId } |
| Tap pay | `createTapCheckout` | None | Returns { success, chargeId, url } |
| Stripe pay | `createStripeCheckout` | None | Returns { success, url, sessionId } |
| Resume payment | `resumeBookingPayment` | None | Re-creates payment for existing booking |
| Calendar sync | `syncBookingToCalendar` | None | Creates Google Calendar event |
| Send confirmation email | `sendBookingConfirmation` | None | Sends booking confirmation email |
| Send status email | `sendBookingStatus` | None | Sends booking status update email |
| Send CV feedback | `sendCVFeedback` | None | Sends CV review feedback email |
| Send invitation | `sendInvitationEmail` | None | Sends call invitation email |
| Send submission response | `sendSubmissionResponse` | None | Sends custom response to applicant |
| Send offer email | `sendOfferEmail` | None | Sends offer (coffee/membership/course) |
| Send password setup | `sendPasswordSetupEmail` | None | Sends password setup email |
| Send test email | `sendTestEmail` | None | Tests SMTP config |
| Send contact reply | `sendContactReply` | None | Replies to contact form |
| Send quotation | `sendQuotation` | None | Sends a quotation email |
| Delete user data | `deleteUserData` | Admin/Manager | Removes all Firestore docs + Auth user |
| Setup admin | `setupAdmin` | Admin key | Creates/repairs admin account |
| Cleanup users | `cleanupSystemUsers` | Admin key | Removes orphaned systemUsers |
| Extract CV data | `extractCVData` | None | AI-extracts CV data |
| Branded password reset | `sendBrandedPasswordReset` | None | Sends branded reset email |

All Cloud Functions run in `us-central1`.

---

## 5. Firestore collections quick reference

| Collection | Purpose | Key status field |
|---|---|---|
| `bookings` | Consultation/meeting bookings | `status`, `paymentStatus` |
| `job_submissions` | CV submissions / candidates | `status` (funnel stage) |
| `student_crm` | Enrolled students | `status`, `callStatus` |
| `call_invitations` | Call invitations | `status` |
| `meeting_invitations` | Meeting links | `status` |
| `contact_submissions` | Website contact form | `status` |
| `user_messages` | Student → admin messages | `status` |
| `communication_status` | Email funnel tracking | `funnelStatus` |
| `enrollments` | Course enrollments | `status`, `progress` |
| `subscriptions` | Subscription plans | `status` |
| `systemUsers` | Admin/portal users | `role` |
| `coupons` | Discount coupons | `isActive` |
| `service_offerings` | Service packages | `isActive` |
| `session_outcomes` | Post-session proof-of-work | `published` |
| `paypal_config` | PayPal credentials | `isActive` |
| `tap_config` | Tap credentials | `isActive` |
| `payment_config` | Booking price + default method | `defaultPrice`, `defaultMethod` |
| `email_logs` | Email send log | `status` |
| `analytics_sessions` | Visitor sessions | — |
| `analytics_events` | Visitor events | — |
| `audit_log` | Admin audit trail | — |
| `shared_links` | Shareable report links | — |
| `slot_overrides` | Calendar slot overrides | `type` |
| `calendar_blocks` | Blocked time periods | — |
| `slot_groups` | Slot groupings | — |
| `price_escalation_rules` | Auto-pricing rules | — |

---

## 6. Payment config document

**Collection:** `payment_config`
**Document:** `booking`

```json
{
  "defaultPrice": 4.50,
  "defaultMethod": "paypal",
  "updatedAt": timestamp
}
```

iOS should read this to know:
- The default booking price (before coupons)
- Which payment method to show first (currently `paypal`)

---

## 7. Key rules for iOS

1. **PayPal is default.** Show PayPal as the primary payment option. Tap/Stripe are secondary.
2. **Always pass `finalPrice`** to `createPayPalOrder` if a coupon is applied — this is the discounted amount.
3. **Poll `bookings/{id}`** after PayPal redirect to confirm `paymentStatus === "paid"`.
4. **Use `paymentMethod` field** — never assume; read it from the booking document.
5. **Funnel stages** are the `status` field on `job_submissions`. Use the stage groups above for pipeline views.
6. **Student CRM** is separate from job submissions. A candidate becomes a student CRM entry when they're invited/enrolled.
7. **User 360** aggregates by email across all collections. Use this for a unified student profile.
8. **Timestamps** from Cloud Functions arrive as `{ _seconds, _nanoseconds }`. Decode with `JSONDecoder`.
9. **`deleteUserData(email)`** is the only safe way to remove a person — it cleans all collections + Firebase Auth.
10. **Role gating:** Read `systemUsers/{uid}.role` to show/hide tabs (`admin` / `manager` / `advisor`).

---

## 8. What's new since last iOS sync

| Change | Impact on iOS |
|---|---|
| PayPal is default payment method | iOS should default to PayPal in booking flow |
| `createPayPalOrder` now accepts `bookingId` + `finalPrice` | iOS can pass existing booking ID and discounted price |
| Admin Payments tab now has PayPal test (not Tap test) | iOS admin payments screen should test PayPal, not Tap |
| `payment_config/booking.defaultMethod` added | iOS can read this to determine which payment to show first |
| PayPal credentials in `paypal_config` Firestore collection | iOS does NOT need credentials — CF handles this |
| Meeting booking page uses `createPayPalOrder` | iOS meeting invitation flow should use `createPayPalOrder` |
| Discover page uses `createPayPalOrder` | iOS discover/booking flow should use `createPayPalOrder` |
| `paypalOrderId` and `paypalTransactionId` on bookings | iOS should read these for PayPal payment verification |

---

*For structural alignment see `IOS_PORTAL_ALIGNMENT_JULY25.md`. For integration roadmap see `WEB_IOS_INTEGRATION_ROADMAP_JULY25.md`.*
