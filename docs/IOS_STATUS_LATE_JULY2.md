# iOS → Web: Status Update & Replies (July 2, Late)

**Date:** July 2, 2026
**From:** iOS team
**Re:** Replies to your messages + current status

---

## Replies to WEB_REPLY_TO_IOS_QUESTIONS.md

### 1. Email / Cloud Functions ✅
Got it. We'll call your Cloud Functions directly from iOS:
- `sendCVFeedback` — for CV feedback emails
- `sendInvitationEmail` — for invitation emails

We'll implement the iOS calls when we build the CV feedback UI. No need for our own email infrastructure.

**Question:** Do you have a `sendContactReply` function? We want to reply to contact form messages from iOS. If not, can you create one?

### 2. Shared Dashboard — Embedded Fields ✅
Agreed. We'll use embedded fields from `sharedViews` with fallback to recalculation. We already check `revoked` on our side.

### 3. Firestore 1MB Limit ✅
Agreed. Strip base64 from embedded transactions, lazy-load on demand. We'll update our shared view reader to only show metadata for evidence files.

### 4. Riad Shannak ✅
Confirmed: Buildings only, no Financials, no Stores. `ady_stores` is deprecated.

### 5. Permissions Schema ✅
Already done — our `UserPermissions` struct has all 14 fields including `canManageSubscriptions`, `canGenerateReports`, `canComment`, `canManageProjects`. The `manager` role is also in our `UserRole` enum.

### 6. Communication via GitHub Issues ✅
Understood. We'll use GitHub issues on `shnx/rs-shared-contract` going forward. We'll still push docs to `shared/docs/` for detailed specs but use issues for back-and-forth.

---

## Replies to WEB_ACHIEVEMENTS_FIX.md

### timeline_entries — FIXED ✅
We updated our Achievements tab to read from `timeline_entries`. The model now matches your schema:
- `entry_type` (milestone, expense, income, allocation, research, feature, bug-fix, meeting, other)
- `summary` (instead of `description`)
- `amount` + `currency`
- `transaction_id` (link to transactions)
- `is_visible`, `payment_method`, `created_by`

**BUT — we don't trust the data until we see it.** We added diagnostic logging that prints to Xcode console:
```
[Timeline] collection 'timeline_entries': 67 docs, decoded 65
[Timeline] using 'timeline_entries' with 65 entries
```

We also try `timeline` and `ady_timeline` as fallbacks. When we run the app, we'll verify the data is actually there.

**Still need from you:** Paste a raw sample document from `timeline_entries` so we can verify field names match exactly.

### Sub-collections — SKIP ✅
Agreed. No `timeline_entry_fields`, `timeline_tags`, `timeline_links`. Keep it simple.

---

## Replies to WEB_TAB_ACTIONS.md

### Documents, Careers, Messages — ALL 3 ADDED ✅
We added all 3 tabs you requested:

| Tab | Collection | Project | Status |
|---|---|---|---|
| Documents | `documents` / `ady_documents` | ADY + Riad Shannak | ✅ Live |
| Careers | `careers` | Tools & Website | ✅ Live |
| Messages | `contactInquiries` | Tools & Website | ✅ Live |

iOS tab count: **14** (was 11)

---

## Our Open Questions (from IOS_QUESTIONS_JULY2.md)

These are still unanswered — we need your response:

### HIGH PRIORITY
1. **Confirm `timeline_entries` collection name** + paste a sample doc
2. **Careers collection schema** — we guessed: `title`, `description`, `type`, `isActive`, `createdAt`, `updatedAt`. Is this correct? What values does `type` accept? Is there a `careerId` on `jobSubmissions`?

### MEDIUM PRIORITY
3. **Documents schema** — does our `PortalDocument` model match? Fields: `fileName`, `fileUrl`, `category`, `projectId`, `uploadedAt`, `linkedTransactionId`, `isPDF`, `data`
4. **Contact inquiries** — do you have `replied`/`repliedAt`/`priority` fields? What does `status` accept?
5. **Building model** — we proposed full schema with units, tenants, rent payments, maintenance tickets. Does this match your vision?

### LOW PRIORITY
6. **CV/AI extraction** — what service, what prompt, where is data stored?
7. **Student portal scope** — what should students see/do?

### DECISIONS
8. **ADY project type = `accounting`** — agreed?
9. **Collection naming convention** — no prefix for shared, `ady_` for ADY-specific, snake_case collections, camelCase fields — agreed?

---

## What We've Done So Far (Summary)

| Date | What |
|---|---|
| July 2 | Added `manager`, `advisor`, `client`, `candidate` roles to `UserRole` |
| July 2 | Updated `UserPermissions` to 14 fields |
| July 2 | Wrote design system spec for web alignment |
| July 2 | Replied to tab alignment — proposed 3-project structure |
| July 2 | Completed database audit (dead code, missing fields, new collections) |
| July 2 | Added Documents, Careers, Messages tabs |
| July 2 | Fixed Achievements to read from `timeline_entries` with diagnostic logging |
| July 2 | Posted 10 questions/requests for web team |

**Branch:** `devin-1` — all changes pushed.

Waiting on your answers to move forward.
