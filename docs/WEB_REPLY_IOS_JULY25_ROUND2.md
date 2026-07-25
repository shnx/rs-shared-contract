# Web Reply — iOS Round 2 Questions — July 25, 2026

Replying to `IOS_REPLY_JULY25_ROUND2.md` questions Q1–Q6 plus screen specs.

---

## Q1: `tap-config` category placement

**Answer: `tap-config` is under `admin` category (Settings group).**

The tab is registered with `category: 'admin'` in `src/tabs/tapConfigTab.tsx`. It appears in the **Settings** group on the web portal, not Operations. The alignment doc listing it under Operations was an error — it should be under **Settings**.

The tab label is **"Payments"** (not "Tap Config") and it manages:
- Default payment method (PayPal or Tap) via `payment_config/booking.defaultMethod`
- Default booking price via `payment_config/booking.defaultPrice`
- Tap gateway credentials (legacy)
- PayPal test payment button

**iOS action:** Keep `tap-config` under **Settings**. Rename to **"Payments"** if not already.

---

## Q2: `paypal_config` admin UI

**Answer: No separate PayPal config screen exists yet.**

PayPal credentials (`paypal_config` collection) are currently managed directly in Firestore — there is no admin UI for adding/editing PayPal credentials. The `TapConfigManagement` page (labeled "Payments") handles Tap credentials and has a **Test PayPal Payment** button, but does not manage PayPal credentials themselves.

The `paypal_config` collection schema:
```
paypal_config/{docId}:
{
  clientId: string,
  clientSecret: string,
  environment: "live" | "sandbox",
  isActive: boolean
}
```

The Cloud Function `createPayPalOrder` reads from this collection:
```
admin.firestore().collection("paypal_config").where("isActive", "==", true).limit(1)
```

**iOS action:** You do NOT need to build a PayPal config UI for now. Credentials are managed via Firestore console. If you want to add one later, it would go under Settings → Payments as a section within the same screen.

---

## Q3: Student Portal — Community tab

**Answer: Backed by `candidate_posts` Firestore collection.**

Schema:
```
candidate_posts/{id}:
{
  id: string,
  authorId: string,         // Firebase Auth UID
  authorName: string,
  authorEmail: string,
  title: string,
  body: string,
  category: "question" | "discussion" | "sharing" | "advice",
  attachments?: MessageAttachment[],
  links?: string[],          // extracted URLs from body
  replies: CandidateReply[], // embedded array
  createdAt: Timestamp
}

CandidateReply:
{
  id: string,
  authorId: string,
  authorName: string,
  authorEmail: string,
  body: string,
  attachments?: MessageAttachment[],
  links?: string[],
  createdAt: Timestamp
}
```

Operations:
- **List:** `candidate_posts` collection, all docs, sorted by `createdAt` desc
- **Create post:** `candidatePostDB.create({ authorId, authorName, authorEmail, title, body, category, links, attachments, replies: [], createdAt })`
- **Add reply:** Read post, append to `replies` array, update doc
- **Delete post:** Delete doc by id (author can delete their own)

**iOS action:** Implement Community as a feed of posts from `candidate_posts` collection. Posts have categories (question, discussion, sharing, advice). Replies are embedded in the post document.

---

## Q4: `resumeBookingPayment` — booking already paid

**Answer: Show booking details, do NOT open Safari.**

When `resumeBookingPayment` returns `{ success: true, url: "${SITE_URL}/book/confirm?booking_id={id}" }` (without a PayPal URL), the booking is already paid. iOS should:
1. Show the booking details / confirmation screen directly
2. Do NOT open the URL in Safari — it's a web-only redirect

When it returns `{ success: true, url: "https://www.paypal.com/checkoutnow?token=..." }`, open that in `SFSafariViewController`.

**iOS action:** Check if `url` contains `paypal.com`. If yes → open Safari. If no → show booking details in-app.

---

## Q5: Booking Preview tab

**Answer: Permanent addition, but LOW priority for iOS.**

`booking-preview` is a permanent admin tab that renders the public `DiscoverPage` inside the admin portal so admins can preview the booking experience. It's useful for QA but not critical for iOS.

**iOS action:** Skip for now. If you build it later, it's a simple WebView or embedded view showing the public booking page. Priority P3.

---

## Q6: `communication_status` collection

**Answer: Per-email, not per-user. Document ID is auto-generated.**

Schema:
```
communication_status/{autoId}:
{
  id: string,                // auto-generated
  email: string,             // the key — normalized to lowercase
  status: "hot" | "follow-up" | "cool" | "do-not-contact" | "ok",
  notes?: string,
  nextAction?: string,       // suggested next step
  updatedAt: Timestamp,
  updatedBy?: string         // admin name
}
```

Operations:
- **Get all:** `firebaseDB.getAll('communication_status')` — sorted by `updatedAt` desc
- **Get by email:** Filter all docs where `email` matches (normalized lowercase)
- **Upsert:** If a doc with the same email exists, update it; otherwise create new

The `upsert` function looks up by email (not by doc ID) and updates if found, creates if not.

**iOS action:** Treat `communication_status` as per-email records. Use email as the logical key. Upsert by querying for existing email, then update or create.

---

## 7. Student Portal screen specs

### Submission tab
- Shows the user's job submission from `job_submissions` collection (filtered by `email === currentUser.email`)
- If no submission: shows empty state with "You haven't submitted an application yet"
- If submission exists: shows status badge, position applied for, submitted date, CV link

### Sessions tab
- Shows bookings from `bookings` collection filtered by `email === currentUser.email`
- Sorted by `scheduledAt` descending
- Each card shows: service name, date/time, status badge (pending/confirmed/completed/cancelled), payment status badge (paid/unpaid), price, payment method
- Unpaid + pending bookings show **"Pay with PayPal"** button (calls `resumeBookingPayment`) and **"Confirm"** link (navigates to `/book/confirm`)
- Empty state: "No sessions booked yet" with CTA to book

### Book Session tab
- Renders `PortalBookingPage` component — same booking flow as public Discover page
- Slot picker (Berlin timezone), service offering selector, coupon input
- PayPal payment via `createPayPalOrder`
- Pre-fills name and email from `currentUser`

### My CV tab
- Shows CV file if user has a submission with CV attached
- Displays file viewer / download link
- If no CV: shows empty state

### Ask Admin tab
- Renders `AskAdmin` component
- User can send a message to admin team
- Stored in `user_messages` collection with `userEmail === currentUser.email`

### Community tab
- Renders `CandidateCommunity` component
- Feed of posts from `candidate_posts` collection
- User can create posts (title, body, category), reply to posts, delete own posts
- Categories: question, discussion, sharing, advice

### Inbox tab
- Renders `StudentInbox` component
- Shows admin replies / notifications for the user
- Read/unread status

### Settings tab
- Renders `StudentSettings` component
- Change username, password
- Account preferences

---

## 8. Client Portal screen specs

### Overview tab
- Stats cards: Upcoming Sessions, Completed Sessions, Documents count, Unread Messages count
- Each card is clickable → navigates to the relevant tab
- Shows client's assigned projects as badges

### Sessions tab
- Same as Student Portal Sessions — bookings filtered by `email === currentUser.email`
- Split into upcoming and past sessions
- Shows status, payment status, date/time, service name

### Documents tab
- Documents from `documents` collection filtered by `projectId` in client's `projectIds`
- Shows document name, type, upload date, download link

### Messages tab
- Messages from `user_messages` collection filtered by `userEmail === currentUser.email`
- Shows unread count badge
- User can send messages to admin

### Settings tab
- Same as Student Portal Settings — change username, password

### Data loading
```
bookings:  bookingDB.getByEmail(currentUser.email)
documents: documentDB.getAll().filter(d => currentUser.projectIds.includes(d.projectId))
messages:  userMessageDB.getAll().filter(m => m.userEmail === currentUser.email)
projects:  firebaseDB.getAll('projects').filter(p => currentUser.projectIds.includes(p.id))
```

---

## 9. Summary of answers

| Q | Answer |
|---|---|
| Q1 | `tap-config` is under **Settings** (category: `admin`). Label is "Payments". |
| Q2 | No PayPal config UI exists. Credentials managed via Firestore `paypal_config` collection. |
| Q3 | Community backed by `candidate_posts` collection. Posts with embedded replies. |
| Q4 | If `resumeBookingPayment` returns non-PayPal URL → show booking details in-app, don't open Safari. |
| Q5 | Booking Preview is permanent but P3 for iOS. Skip for now. |
| Q6 | `communication_status` is per-email, auto-generated doc ID, upsert by email lookup. |

---

*Pushed by web team — July 25, 2026.*
