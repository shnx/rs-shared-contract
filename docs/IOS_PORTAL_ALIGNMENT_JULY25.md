# iOS ↔ Web Portal Alignment — July 25, 2026

The iOS admin app must mirror the web portal structure. This doc maps every web portal category and tab to its iOS equivalent screen so the iOS team stops building ad-hoc flows and follows the web as the source of truth.

---

## 0. Alignment rule

**Web is the source of truth for structure and behavior.** iOS should copy the same categories, tabs, and data shapes. Do not invent new screens or rename concepts. If a feature exists on web, iOS uses the same Firestore collections and Cloud Functions.

---

## 1. Web portal categories → iOS bottom/side nav

| Web category | iOS screen group | Required tabs/screens |
|---|---|---|
| **Dashboard** | Dashboard | documents, course-overview (education), riad-portal (real_estate), timeline (accounting) |
| **People** | People | funnel-manager, user-360 |
| **Bookings & Services** | Bookings | calendar, invitations-coupons, service-offerings, session-outcomes, price-escalation, **booking-preview** (new) |
| **Operations** | Operations | buildings (real_estate), proposal-generator, communication, user-messages, analytics, tap-config |
| **Website** | Website | website-manager |
| **Settings** | Settings | projects, user-management, admin-settings, link-manager |

*Note: Finance (transactions, budget, reports) and CRM are web-only for now unless iOS is explicitly asked to build them.*

---

## 2. Screen-by-screen requirements

### Dashboard
- `documents`: upload, view, PDF extraction, Drive sync.
- `course-overview`: show education/client progress cards.
- `riad-portal`: real estate project dashboard.
- `timeline`: chronological project milestones.

### People
- `funnel-manager`: list of all candidates/clients/submissions with filters and actions (invite, call, email, status change).
- `user-360`: search by email, show unified profile with bookings, submissions, messages, emails, calls.

### Bookings
- `calendar`: visual calendar, availability, slot overrides, booking list.
- `invitations-coupons`: create/share booking links, manage coupons.
- `service-offerings`: manage mentorship / call / MVP packages.
- `session-outcomes`: publish proof-of-work after sessions.
- `price-escalation`: view/edit auto-escalation rules.

### Operations
- `communication`: email logs, funnel status, portal messages (new Email Funnel feature).
- `user-messages`: admin inbox for student/candidate messages.
- `analytics`: visitor sessions and events.
- `tap-config`: Tap gateway settings.

### Website
- `website-manager`: contact messages, career opportunities.

### Settings
- `projects`: project selector and management.
- `user-management`: users, candidates, roles.
- `admin-settings`: audit log.
- `link-manager`: shareable report links.

---

## 2.5 Admin action matrix — what iOS must mirror from the web portal

The following table is the source of truth for the iOS admin app. Every web admin feature that reads or writes data is listed here with the Firestore collections, Cloud Functions, and write actions the iOS screen must support.

### Dashboard
| Web tab | iOS screen | Collections | Read actions | Write actions | Cloud Functions |
|---|---|---|---|---|---|
| `documents` | Documents | `documents` | list, preview, search | upload, delete | *(client upload via Firebase Storage or direct Firestore create)* |
| `course-overview` | Course Overview | `student_crm`, `service_offerings`, `bookings` | enrolled clients, progress, upcoming sessions | update progress/notes | *(direct Firestore writes)* |
| `riad-portal` | Riad Portal | `buildings` | buildings, units, occupancy | add/edit buildings | *(direct Firestore writes)* |
| `timeline` | Timeline | `timeline`, `transactions`, `budget` | chronological milestones, transactions | add/edit entries | *(direct Firestore writes)* |

### People
| Web tab | iOS screen | Collections | Read actions | Write actions | Cloud Functions |
|---|---|---|---|---|---|
| `funnel-manager` | Funnel | `job_submissions`, `crm_contacts`, `student_crm`, `call_invitations`, `systemUsers`, `email_logs` | list candidates, filter, search, view email history | send email, invite to call, change status, delete all data | `sendInvitationEmail`, `sendCVFeedback`, `sendSubmissionResponse`, `deleteUserData` |
| `user-360` | User 360 | `systemUsers`, `job_submissions`, `crm_contacts`, `student_crm`, `bookings`, `call_invitations`, `portal_messages`, `email_logs`, `analytics_sessions`, `analytics_events` | unified profile by email | send email, delete all data, impersonate | `sendSubmissionResponseEmail` *(client call)*, `deleteUserData` |

### Bookings & Services
| Web tab | iOS screen | Collections | Read actions | Write actions | Cloud Functions |
|---|---|---|---|---|---|
| `calendar` | Calendar | `bookings`, `slot_overrides`, `calendar_blocks`, `availability`, `slot_groups`, `service_offerings` | calendar view, **upcoming list**, day slots, day bookings | create booking, update status, reschedule, cancel, block/unblock slots, add/remove overrides | `sendBookingStatusEmail`, `sendBookingConfirmationEmail`, `syncBookingToCalendar` |
| `invitations-coupons` | Invitations & Coupons | `coupons`, `meeting_invitations` | list, filter | create coupon, create meeting invitation, toggle active | *(direct Firestore writes)* |
| `service-offerings` | Service Offerings | `service_offerings` | list packages | create, edit, toggle active | *(direct Firestore writes)* |
| `session-outcomes` | Session Outcomes | `session_outcomes`, `bookings` | list outcomes, drafts | publish, edit, unpublish | *(direct Firestore writes)* |
| `price-escalation` | Price Escalation | `price_escalation_rules` | view rules | add/edit/delete rules | *(direct Firestore writes)* |

### Operations
| Web tab | iOS screen | Collections | Read actions | Write actions | Cloud Functions |
|---|---|---|---|---|---|
| `buildings` | Buildings | `buildings` | list buildings/units | CRUD buildings | *(direct Firestore writes)* |
| `proposal-generator` | Proposal Generator | `service_offerings`, `bookings` | generate proposals | create/send quotation | `sendQuotation` |
| `communication` | Communication | `email_logs`, `communication_status`, `portal_messages` | email log per user, funnel status (hot/follow-up/cool/ok), portal messages | set manual funnel status, notes, next action, send test email | `sendTestEmail`, `deleteUserData` *(when deleting a user from any list)* |
| `user-messages` | Messages | `user_messages` | admin inbox | reply, mark read, archive | *(direct Firestore writes)* |
| `analytics` | Analytics | `analytics_sessions`, `analytics_events` | sessions, events | *(read-only for now)* | — |
| `tap-config` | Payments | `tap_config`, `site_config` | view gateway settings | update credentials/settings | *(direct Firestore writes)* |

### Website
| Web tab | iOS screen | Collections | Read actions | Write actions | Cloud Functions |
|---|---|---|---|---|---|
| `website-manager` | Website Manager | `contact_submissions`, `career_opportunities` | inquiries, career applications | mark replied, archive, delete | `sendContactReply` |

### Settings
| Web tab | iOS screen | Collections | Read actions | Write actions | Cloud Functions |
|---|---|---|---|---|---|
| `projects` | Projects | `projects`, `project_members` | list, switch project | create, edit, archive, assign members | *(direct Firestore writes)* |
| `user-management` | User Management | `systemUsers`, `project_members`, `bookings`, `job_submissions` | list users, candidates, roles; show bookings count, paid/unpaid, submission status per user | add user, edit role, assign projects, delete all data, **reset password** | `setupAdmin`, `cleanupSystemUsers`, `deleteUserData`, `sendBrandedPasswordReset` |
| `admin-settings` | Admin | `audit_log` | audit log | *(read-only)* | — |
| `link-manager` | Shared Links | `shared_links` | list links, visits, revoke | create, revoke | *(direct Firestore writes)* |

### Cross-cutting rules for iOS
- **Role gating:** iOS must use `systemUsers/{uid}.role` to show/hide the same tabs as `tabRegistry.ts` (`admin` / `manager` / `advisor`).
- **Student/candidate/graduate routing:** These roles do NOT get the admin portal. They get a **Student Portal** with only: Submission, Sessions, Book Session, My CV, Ask Admin, Community, Inbox, Settings. No analytics, no finance, no admin tabs. See `IOS_UPDATE_JULY25_PORTAL_CHANGES.md`.
- **Client routing:** `client` role gets a **Client Portal** with: Overview, Sessions, Documents, Messages, Settings. No admin access.
- **Project scoping:** Most admin tabs now appear only for `website` projects (`funnel-manager`, `user-360`, `communication`, `calendar`, etc.). Finance tabs appear only for `accounting`. Real estate tabs appear only for `real_estate`.
- **User deletion:** `deleteUserData(email)` is the canonical way to remove a user from all collections + Firebase Auth. Call it from `User 360`, `Funnel Manager`, or `User Management`.
- **Timestamps:** Firestore timestamps from CFs arrive as `{ _seconds, _nanoseconds }`. Decode with Swift `JSONDecoder` and convert to `Date`.
- **PayPal is primary:** Use `createPayPalOrder` as the primary payment method. `createTapCheckout` and `createStripeCheckout` are legacy. Use `resumeBookingPayment` for unpaid bookings in student portal.
- **No auto emails:** Booking confirmation no longer auto-sends emails. Admins can send manually from Communication or Booking Settings.
- **Password reset:** Admin can trigger `sendBrandedPasswordReset(email)` from User Management. Reset URL format: `https://the-rs.com/reset-password?mode=resetPassword&oobCode={code}&lang=en`.

## 3. iOS must not do this

- Do not create separate `Submissions`, `CRM`, `Email Inbox`, `Student Overview` screens — these are merged into `funnel-manager` and `user-360` on web.
- Do not duplicate `calendar` and `invitations-coupons` as separate apps; they are tabs in one `Bookings` group.
- Do not build a different payment flow. Use the same CFs: `createTapCheckout`, `createStripeCheckout`, `createPayPalOrder`.
- Do not store local-only data for features that live in Firestore (statuses, notes, bookings).

---

## 4. Recommended iOS cleanup checklist

1. Audit every iOS screen. If it does not map to a web tab above, delete or merge it.
2. Use the same collection names and field names as `src/types/index.ts`.
3. Replace custom admin navigation with the 4–6 web categories.
4. For public booking/search, use the exact CF sequence in `WEB_IOS_INTEGRATION_ROADMAP_JULY25.md`.

---

*For exact payloads and flows see `WEB_REPLY_IOS_JULY25.md` and `WEB_IOS_INTEGRATION_ROADMAP_JULY25.md`.*
