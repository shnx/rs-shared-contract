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
| **Bookings & Services** | Bookings | calendar, invitations-coupons, service-offerings, session-outcomes, price-escalation |
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
