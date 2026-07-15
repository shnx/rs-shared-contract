# System Architecture & iOS Knowledge Transfer

> **Purpose**: Complete documentation of the web platform's architecture for the iOS team.
> Covers: Discover/Availability, Tap Payments, Messaging, Roles, and Firestore schema.

---

## 1. Discover Page & Availability System

### How It Works

The Discover page (`/discover` or `/book`) is a **multi-step wizard** that works without authentication:

```
Welcome (3 paths) → Packages → Details → Schedule → Confirm → Payment (Tap)
     ↓
  Robotics    Mentorship    Company
     ↓            ↓            ↓
  Quote form   Coffee chat   Lead form
               → Mentorship
```

### What Controls Availability

**Firestore collection: `availability`**

Each document represents a weekly recurring slot:

```typescript
{
  id: string;
  dayOfWeek: number;      // 0 = Sunday, 6 = Saturday
  startTime: string;      // "10:00"
  endTime: string;        // "18:00"
  bufferMinutes: number;  // 15 (gap between slots)
  maxBookingsPerDay: number; // 5
}
```

**Admin manages these** via `AvailabilityManagement.tsx` in the portal (Settings → Availability).

**Slot generation logic** (client-side in `DiscoverPage.tsx`):
1. Find availability for the selected day of week
2. Generate slots from `startTime` to `endTime` with `durationMinutes + bufferMinutes` step
3. Exclude slots that already have bookings (checks `bookings` collection)
4. Respect `maxBookingsPerDay` limit

### How iOS Should Sync

**Read from Firestore directly** (no API needed):

```swift
// Listen to availability changes in real-time
db.collection("availability").addSnapshotListener { snapshot, error in
    // Parse each doc into Availability model
}

// Listen to bookings (to know which slots are taken)
db.collection("bookings")
    .whereField("status", isNotEqualTo: "cancelled")
    .addSnapshotListener { snapshot, error in
    // Filter by date to find taken slots
}
```

**Service offerings** (packages/prices):

```swift
db.collection("service_offerings")
    .whereField("isActive", isEqualTo: true)
    .getDocuments { snapshot, error in
    // Parse into ServiceOffering models
}
```

### Firestore Collections for Discover

| Collection | Read Access | Write Access |
|---|---|---|
| `service_offerings` | Public | Auth only |
| `availability` | Public | Auth only |
| `bookings` | Public read, public create | Update/delete auth only |
| `contact_submissions` | Auth read | Public create |
| `site_config` | Public read | Auth write |

---

## 2. Tap Payment Flow (End-to-End)

### Architecture

```
User selects service + date + time
  → Fills name/email on Discover page
  → Clicks "Continue to Payment"
  → Frontend calls Cloud Function `createTapCheckout`
  → Cloud Function:
      1. Creates pending booking in Firestore
      2. Calls Tap API: POST https://api.tap.company/v2/charges
      3. Returns Tap checkout URL
  → Frontend redirects to Tap checkout page (window.location.href = url)
  → User pays on Tap's hosted page
  → Tap redirects back to: /book/confirm?booking_id=xxx&tap_id=xxx
  → Tap also sends webhook to Cloud Function `tapWebhook`
  → Webhook updates booking: paymentStatus=paid, status=confirmed
  → Sends confirmation email
  → BookingConfirmPage verifies payment, syncs calendar, shows success
```

### Cloud Functions

| Function | Type | Purpose |
|---|---|---|
| `createTapCheckout` | Callable | Creates Tap charge, returns checkout URL |
| `tapWebhook` | HTTP | Receives Tap webhook, confirms payment |

### Configuration Required

1. **Set Tap secret key** in Google Cloud:
```bash
firebase functions:secrets:set TAP_SECRET_KEY
# Enter your Tap secret key (sk_test_... or sk_live_...)
```

2. **Set SITE_URL** in functions env:
```bash
firebase functions:config:set site.url="https://your-domain.com"
# Or set SITE_URL env var
```

3. **Configure webhook URL in Tap Dashboard**:
```
https://us-central1-robotics-website-5593f.cloudfunctions.net/tapWebhook
```

4. **Tap Config Management** (admin portal):
- Admin can also manage Tap API keys via `TapConfigManagement.tsx`
- Keys stored in Firestore `tap_config` collection

### Booking Data Model

```typescript
interface Booking {
  id: string;
  userId?: string;        // Linked if user has account
  email: string;
  name: string;
  company?: string;
  serviceOfferingId: string;
  serviceType: string;
  scheduledAt: Date;
  durationMinutes: number;
  status: 'pending' | 'confirmed' | 'completed' | 'cancelled';
  priceUSD: number;
  tapChargeId?: string;   // Tap charge ID
  paymentStatus: 'unpaid' | 'paid';
  notes?: string;
  createdAt: Date;
}
```

### iOS Implementation

For iOS, you have two options:

**Option A: Use Tap's iOS SDK** (recommended)
```swift
// 1. Call createTapCheckout Cloud Function
let result = try await functions.httpsCallable("createTapCheckout")
    .call(["serviceOfferingId": offeringId, "bookingDetails": details])

// 2. Get checkout URL from result
let url = result.data["url"] as! String

// 3. Open in SafariViewController or Tap SDK
let safari = SFSafariViewController(url: URL(string: url)!)
present(safari, animated: true)

// 4. Listen to bookings collection for payment confirmation
db.collection("bookings").document(bookingId)
    .addSnapshotListener { doc, _ in
        if doc?.data?["paymentStatus"] == "paid" {
            // Show success screen
        }
    }
```

**Option B: Deep link back to app**
- Set redirect URL to a custom scheme: `roboticsciences://book/confirm?booking_id=xxx`
- Configure universal links in Xcode

---

## 3. Messaging System

### Overview

There are **two messaging systems**:

### A. Portal Messages (In-App)

**Admin → Users** announcements within the portal.

**Firestore collection: `portal_messages`**

```typescript
interface PortalMessage {
  id: string;
  senderId: string;
  senderName: string;
  title: string;
  body: string;
  bodyAr?: string;         // Arabic version
  audience: 'all' | 'students' | 'clients' | 'candidates' | 'specific';
  targetEmails?: string[]; // When audience = 'specific'
  priority: 'normal' | 'important' | 'urgent';
  isRead: boolean;
  readBy: string[];        // Array of user emails who read it
  createdAt: Date;
  expiresAt?: Date;
}
```

**Who gets what:**

| Audience | Who sees it |
|---|---|
| `all` | Every authenticated portal user |
| `students` | Users with `role = 'student'` |
| `clients` | Users with `role = 'client'` |
| `candidates` | Users with `role = 'candidate'` |
| `specific` | Only users whose email is in `targetEmails[]` |

**Admin sends** via `MessagingCenter.tsx` (portal → Messaging tab).
**Users see** via `StudentInbox.tsx` (portal → Inbox).

**Filtering logic** (in `portalMessageDB.getByAudience`):
```typescript
if (m.audience === 'all') return true;
if (m.audience === 'students' && userRole === 'student') return true;
if (m.audience === 'clients' && userRole === 'client') return true;
if (m.audience === 'candidates' && userRole === 'candidate') return true;
if (m.audience === 'specific' && userEmail && m.targetEmails?.includes(userEmail)) return true;
```

### B. Email System (External)

**Admin → Applicant** emails sent via Cloud Functions.

| Cloud Function | Trigger | Purpose |
|---|---|---|
| `sendSubmissionResponse` | Callable (admin) | Send custom HTML email to applicant |
| `sendOfferEmail` | Callable (admin) | Send offer email with course/coffee/membership options |
| `sendCVFeedback` | Callable (admin) | Send CV feedback with registration link |
| `sendInvitationEmail` | Callable (admin) | Send call invitation with registration link |
| `sendBookingConfirmation` | Callable | Send booking confirmation after payment |
| `sendBookingStatus` | Callable | Send booking status update (confirmed/cancelled/completed) |
| `sendQuotation` | Callable (admin) | Send price quotation to company leads |
| `tapWebhook` | HTTP webhook | Auto-send confirmation after Tap payment |

**Email provider**: Resend API (`https://api.resend.com/emails`)
**From**: `"Robotics Sciences" <info@the-rs.com>`
**Reply-to**: `info@the-rs.com`

### C. Email Response Capture

Students can respond to emails via buttons that link to:
```
/email-response?token=xxx&action=accept|decline|survey
```

This hits the `captureEmailResponse` HTTP function which:
1. Finds CRM contact by `responseToken`
2. Records the response (accept/decline/survey answers)
3. Updates CRM contact status

### iOS Messaging Implementation

```swift
// Listen to portal messages for current user
db.collection("portal_messages")
    .whereField("audience", in: ["all", userRole])
    .addSnapshotListener { snapshot, _ in
        // Also check "specific" messages that include user's email
        // Filter by expiry date
        // Show unread badge count
    }

// Mark as read
db.collection("portal_messages").document(msgId)
    .updateData(["readBy": FieldValue.arrayUnion([userEmail])])
```

---

## 4. Roles & Permissions

### User Roles

Defined in `src/types/index.ts`:

```typescript
type UserRole = 'admin' | 'manager' | 'student' | 'client' | 'advisor' | 'candidate';
```

### Role Hierarchy & Permissions

| Permission | admin | manager | advisor | client | student | candidate |
|---|---|---|---|---|---|---|
| View Dashboard | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| View Financials | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ |
| Project Management | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| View CRM | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ |
| View Contracts | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ |
| View Transactions | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ |
| View Reports | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ |
| Manage Users | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Edit | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| Delete | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |

### Firestore: `systemUsers` Collection

```typescript
interface User {
  id: string;
  name: string;
  role: UserRole;
  email?: string;
  username?: string;
  permissions: { /* see permissions table above */ };
  projectIds: string[];
  createdAt: Date;
}
```

### Role Assignment Flow

```
1. Student submits CV → /join-us
   → Created in `job_submissions` (no account yet)
   → Role is implicitly "applicant" (no systemUsers doc)

2. Admin reviews submission → SubmissionResponse.tsx
   → Admin clicks "Create Account"
   → Creates Firebase Auth user (email/password)
   → Creates systemUsers doc with role = 'candidate'
   → Sends email with login instructions

3. Student logs in for the first time
   → onAuthStateChanged fires
   → linkCandidateToSubmission() runs
   → Finds matching submission by email
   → Creates quiz if none exists
   → Sets role to 'candidate' if user has no role

4. Admin upgrades role
   → User Management tab in portal
   → Can change role from 'candidate' → 'student' → 'advisor' etc.
   → Updates systemUsers doc in Firestore
```

### Admin Email Whitelist

Hardcoded in `AuthContext.tsx`:
```typescript
const ALLOWED_EMAILS = ['mohammedshannak@gmail.com'];
```

**Login gatekeeper** checks:
1. Is email in admin whitelist? → Allow
2. Does email have a `systemUsers` profile? → Allow
3. Does email have a `job_submissions` record? → Allow
4. None of the above? → Reject and sign out

### iOS Role Sync

```swift
// After Firebase Auth login, fetch user profile
db.collection("systemUsers").document(authUser.uid)
    .getDocument { doc, _ in
        let role = doc?.data?["role"] as? String ?? "candidate"
        let permissions = doc?.data?["permissions"] as? [String: Bool]
        
        // Store in app state
        AppState.shared.userRole = UserRole(rawValue: role)
        AppState.shared.permissions = permissions
        
        // Show appropriate tabs/views based on role
    }

// Listen for role changes in real-time
db.collection("systemUsers").document(authUser.uid)
    .addSnapshotListener { doc, _ in
        // Update UI if admin changes role
    }
```

### Admin Controls

Admin can (via portal):
- **User Management**: Change any user's role, edit permissions, deactivate
- **Messaging Center**: Send announcements to targeted audiences
- **Submission Response**: Review CVs, create accounts, send offers
- **Availability Management**: Set weekly available slots
- **Service Offerings**: Create/edit/disable service packages
- **Tap Config**: Manage payment gateway API keys
- **Bookings Management**: View all bookings, update status, cancel

---

## 5. Firestore Collections Reference

| Collection | Purpose | Public Access |
|---|---|---|
| `systemUsers` | User profiles with roles | Auth only |
| `job_submissions` | CV submissions from /join-us | Public create, auth read |
| `contact_submissions` | Company inquiries | Public create, auth read |
| `service_offerings` | Service packages/pricing | Public read |
| `availability` | Weekly time slots | Public read |
| `bookings` | Booking records | Public read + create |
| `tap_config` | Tap API keys | Auth only |
| `portal_messages` | In-app announcements | Auth only |
| `student_crm` | Student CRM contacts | Auth only |
| `call_invitations` | Call invite records | Auth only |
| `crm_contacts` | Email response tracking | Auth only |
| `quotation_requests` | Company quote requests | Auth only |
| `site_config` | Feature flags, settings | Public read |
| `sharedViews` | Shareable dashboard links | Public read |
| `venture_reports` | Public report pages | Public read |
| `timeline` | Achievement photos | Public read |

---

## 6. Cloud Functions Reference

| Function | Type | Auth Required | Purpose |
|---|---|---|---|
| `createTapCheckout` | Callable | No | Create Tap payment session |
| `tapWebhook` | HTTP | No (webhook) | Process Tap payment callback |
| `sendSubmissionResponse` | Callable | Yes | Send custom email to applicant |
| `sendOfferEmail` | Callable | Yes | Send offer email with options |
| `sendCVFeedback` | Callable | Yes | Send CV feedback email |
| `sendInvitationEmail` | Callable | Yes | Send call invitation |
| `sendBookingConfirmation` | Callable | No | Send booking confirmed email |
| `sendBookingStatus` | Callable | No | Send booking status update |
| `sendQuotation` | Callable | Yes | Send price quotation |
| `captureEmailResponse` | HTTP | No | Capture email button responses |
| `syncBookingToCalendar` | Callable | No | Sync booking to Google Calendar |
| `setupAdmin` | Callable | No (setup key) | One-time admin setup |

---

## 7. iOS Sync Summary

### What iOS Should Do

1. **Authentication**: Use Firebase Auth (Google Sign-In, Email/Password, Apple Sign-In)
2. **User Profile**: After login, fetch `systemUsers/{uid}` for role + permissions
3. **Discover/Booking**: Read `service_offerings` + `availability` + `bookings` directly from Firestore
4. **Payments**: Call `createTapCheckout` Cloud Function → open Tap checkout URL → listen to `bookings/{id}` for confirmation
5. **Messaging**: Listen to `portal_messages` filtered by audience/role
6. **CV Submission**: Write to `job_submissions` (public, no auth needed)
7. **Real-time updates**: Use Firestore snapshot listeners for bookings, messages, user profile changes

### What iOS Should NOT Do

- Do NOT store Tap secret key in the app — always use Cloud Function
- Do NOT write bookings directly — use `createTapCheckout` Cloud Function
- Do NOT manage roles client-side — read from `systemUsers` and listen for changes
- Do NOT send emails from iOS — use Cloud Functions

### Shared Types

Create Swift models matching these TypeScript interfaces:
- `User` (with `UserRole` enum)
- `ServiceOffering`
- `Availability`
- `Booking`
- `PortalMessage`
- `JobSubmission`
- `StudentCRMContact`

All field names in Firestore are camelCase. Date fields are stored as Firestore Timestamps.
