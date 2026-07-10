# iOS → Web: Sync Response — July 10

**From:** iOS team
**To:** Web team
**Date:** July 10, 2026
**Re:** WEB_IOS_SYNC_REQUEST_JULY9.md — responding to all action items

---

## 1. ✅ 3-Project Structure Confirmed

The iOS app now shows exactly **3 projects**, matching the web team's decision:

| # | Project | iOS Key | Tabs |
|---|---------|---------|------|
| 1 | ADY | `ady` | Dashboard, Budget, Transactions, Payments, **Insights** (new), Timeline, Documents |
| 2 | Website Tools | `websiteTools` | Submissions, My Track, CRM, LaTeX, Carousel, Careers, Messages |
| 3 | Riad Shannak | `riadShannak` | Buildings, Documents |

- Removed `studentTrack` project (merged into Website Tools as "My Track" tab)
- ADY removed from main tab bar — accessed only through Projects hub
- Projects hub shows 3 hardcoded cards instead of loading from Firestore `projects` (which was showing untitled/duplicate projects)
- No references to `internal`, `education`, `consulting`, or `client_interface` as project types

---

## 2. iOS Firebase Collections (Complete List by Tab)

### Main Tab Bar (Admin)
| Tab | Collections | Read/Write |
|-----|-------------|------------|
| Projects | `projects` (read-only, for hub cards) | Read |
| Screenshot | None (local PhotosPicker + UserDefaults) | Local only |
| LaTeX | None (local compilation) | Local only |
| Carousel | `gemini` API only | API only |
| Roles | `systemUsers`, `employee_permissions` | Read/Write |

### ADY Project Tabs
| Tab | Collections | Read/Write |
|-----|-------------|------------|
| Dashboard | `ady_transactions`, `ady_periods`, `periodSummaries`, `projects` | Read |
| Budget | `ady_periods`, `ady_transactions`, `ady_documents`, `periodSummaries` | Read/Write |
| Transactions | `ady_transactions`, `transactions` (fallback) | Read/Write |
| Payments | `ady_transactions` | Read/Write |
| **Insights** (new) | `ady_transactions`, `transactions` (fallback), `systemUsers` (employees) | Read/Write |
| Timeline | `timeline`, `ady_transactions` (for linkage) | Read |
| Documents | `ady_documents`, `documents` | Read/Write |

### Website Tools Project Tabs
| Tab | Collections | Read/Write |
|-----|-------------|------------|
| Submissions | `job_submissions`, `contact_submissions` | Read/Write |
| My Track | `courses`, `enrollments`, `video_tutorials`, `video_assignments`, `student_memberships` | Read |
| CRM | `ady_crm`, `crm_contacts` | Read/Write |
| LaTeX | None | Local only |
| Carousel | Gemini API | API only |
| Careers | `career_opportunities` | Read |
| Messages | `contact_submissions` | Read/Write |

### Riad Shannak Project Tabs
| Tab | Collections | Read/Write |
|-----|-------------|------------|
| Buildings | `buildings`, `ady_transactions` (for property transactions) | Read/Write |
| Documents | `documents`, `ady_documents` | Read/Write |

### Settings
| Section | Collections | Read/Write |
|---------|-------------|------------|
| User Profile | `systemUsers` | Read/Write |
| Manage Users | `systemUsers` | Read/Write |
| Employee Management | `systemUsers` (role=employee), `work_sessions`, `employee_permissions` | Read/Write |
| Auth | Firebase Auth + `systemUsers` | Read/Write |

---

## 3. Features We Built That You May Not Know About

### ADY Insights Tab (NEW — July 10)
- **Category breakdown** — groups expenses using `CategoryEngine.consolidatedBreakdown` into 6 broad categories with progress bars
- **Recurring payments** — lists all transactions where `isRecurring == true`, with recipient, frequency, amount
- **Add recurring payment** — form to create recurring transaction (salary, subscription, etc.) with employee picker
- **AI detect recurring** — sends transaction summaries to Gemini 2.5 Flash, marks detected recurring transactions
- **Collections:** `ady_transactions`, `systemUsers` (for employee picker)
- **No new collections or schema changes** — uses existing Transaction fields

### Screenshot Tool (Rewritten — July 10)
- User picks base image from photo library via `PhotosPicker`
- Draggable overlay placeholders with per-field font/size/color/alignment controls
- AI field detection using Gemini 2.5 Flash Vision to auto-identify sensitive fields
- Preset save/load with full overlay data (stored in UserDefaults)
- Export at original image resolution
- **Collections:** None — entirely local + Gemini API

### Project Hub (New — July 9)
- 3 hardcoded project cards instead of Firestore-loaded projects
- Eliminates the "untitled projects" issue from duplicate Firestore project docs
- **Collections:** None (hardcoded enum)

---

## 4. Discover Page Alignment

**Status:** iOS does not currently have a Discover page equivalent.

We are not using `service_offerings`, `availability`, `bookings`, or `quotations` collections on iOS.

**Recommendation:** The Discover/booking system is web-only for now. If iOS needs it later, we'll align with your schemas. No action needed from either team at this time.

---

## 5. Duplicate Collections / Naming Conflicts

### Known dual-collections (iOS reads both, web writes unprefixed):
| iOS reads | Web writes | Notes |
|-----------|-----------|-------|
| `ady_transactions` + `transactions` | `transactions` | iOS reads both, merges by ID + `updatedAt` |
| `ady_periods` + `periods` | `periods` | Same merge logic |
| `ady_documents` + `documents` | `documents` | Same merge logic |

**No conflicts found.** The prefix logic is working as designed — iOS writes to `ady_*` collections, web writes to unprefixed, both read both.

### Collections iOS uses that web may not know about:
| Collection | Purpose | Status |
|-----------|---------|--------|
| `work_sessions` | Employee time tracking | iOS-only |
| `employee_permissions` | Employee access control | iOS-only |
| `timeline` | Project timeline/achievements | Shared (web has `timeline_entries` — possible naming mismatch?) |

**⚠️ Possible naming mismatch:** iOS uses `timeline` but web docs mention `timeline_entries`. Please confirm which is canonical.

---

## 6. Gemini Model Update

**All iOS Gemini calls updated from `gemini-2.0-flash` to `gemini-2.5-flash`.**

Gemini 2.0 Flash was shut down by Google on June 1, 2026. If the web team is using Gemini, please update to `gemini-2.5-flash` as well.

---

## 7. Main Tab Bar Cleanup (July 10)

Removed from admin main tabs:
- **Submissions** — moved to Website Tools project only
- **Achievements** — removed entirely (no value)
- **Report Generator** — removed (broken, not generating proper reports)

Admin main tabs are now: **Projects, Screenshot, LaTeX, Carousel, Roles**

---

## 8. Summary

| Item | Status |
|------|--------|
| 3-project structure | ✅ Confirmed and implemented |
| Collections audit | ✅ Provided above |
| Hidden features shared | ✅ Insights tab, Screenshot tool, Project hub |
| Discover page | N/A — iOS doesn't have it, no conflict |
| Duplicate collections | ✅ No conflicts, prefix logic working |
| Gemini model | ✅ Updated to 2.5-flash |
| `timeline` vs `timeline_entries` | ⚠️ Needs confirmation from web team |

**No Firestore schema changes from iOS side.** All new features use existing fields and collections.
