# Web → iOS: Major Portal Fixes + Project Alignment (July 2, Evening)

**Date:** July 2, 2026, 10:20pm UTC+2
**From:** Web team (Mohammad Shannak)
**Re:** ADY project fixed, project picker forced, achievements working, tab alignment updates

---

## 1. ADY Project Was Broken — Now Fixed ✅

**Root cause:** The ADY project in Firestore had `type: undefined`. Since every tab checks `projectTypes.includes(projectType)`, an undefined type matched **zero tabs**. ADY appeared empty with no pages.

**What we did:**
- Set ADY's `type` to `'accounting'` in Firestore
- Set `attachedTabs` to 11 tabs: dashboard, budget, transactions, reports, achievements, documents, settings, launchpad, finance, charts, project-management
- Set description: "ADY — Applied Data & Youth venture. Financial tracking, transactions, achievements, and reports."

**ADY now has real data visible:**
| Collection | Docs | Status |
|---|---|---|
| `ady_transactions` | 66 | ✅ Working |
| `ady_periods` | 5 | ✅ Working |
| `timeline_entries` | 67 | ✅ Working (57 expenses, 3 income, 7 milestones) |

All 67 timeline_entries belong to ADY (`project_id: 1764173848951kdyjg245d`). Most are auto-created from transactions with `transaction_id` links.

---

## 2. Achievements = timeline_entries (CONFIRMED)

This is the same collection iOS should read from. We already posted `WEB_ACHIEVEMENTS_FIX.md` about this. To recap:

- **Collection:** `timeline_entries` (67 docs) — NOT `timeline` (0 docs)
- **Schema:** `id, title, summary, date (string), entry_type, is_visible, amount?, currency?, category?, payment_method?, transaction_id?, project_id, created_by, created_at, updated_at`
- **entry_type values:** `milestone | expense | income | allocation | research | feature | bug-fix | meeting | other`
- **57 are expenses, 3 are income, 7 are milestones**
- Most financial entries have `transaction_id` linking back to the original transaction in `ady_transactions`

**Transaction → Timeline auto-sync is restored:**
- Creating a transaction → auto-creates a timeline entry
- Updating a transaction → updates the linked timeline entry
- Deleting a transaction → deletes the linked timeline entry

---

## 3. Project Picker Now Forced on Login

Previously, `ProjectContext` auto-selected a project on login, skipping the picker. We removed that — users now **must choose a project** every time they log in.

**Flow:** Login → ProjectSelector (pick project) → Launchpad (with stats + tabs)

### ProjectSelector redesign:
- Dark hero with "Welcome back, {name}" greeting
- Each card shows: type badge, status pill, **tab count** (how many pages available)
- Search bar to filter projects

### Launchpad improvements:
- Quick stats bar in hero: Revenue, Costs, Net, Periods (loaded from Firestore)
- Project switcher dropdown in hero (switch without going back to picker)

---

## 4. Cleaned Up Projects in Firestore

We had 5 projects, 2 were junk:

| Project | Action | Reason |
|---|---|---|
| **ADY** (`1764173848951kdyjg245d`) | ✅ Fixed | Type was undefined, tabs empty — now `accounting` with 11 tabs |
| **Main Website** (`1782035493659ycblbrsnu`) | ✅ Kept | Type `website`, tabs: dashboard, settings, website |
| **Riad Shannak** (`1783017550863vq1y6omkl`) | ✅ Kept | Type `real_estate`, tabs: dashboard, buildings, documents, reports, settings |
| ~~Riad Shannak (duplicate)~~ (`1781983501349sh7smj5w2`) | 🗑️ Deleted | Had `type: accounting`, empty tabs — duplicate of the real_estate one |
| ~~(unnamed)~~ (`90A123F1-...`) | 🗑️ Deleted | No name, `status: planning`, probably created from iOS test |

### Final 3 projects (aligned with iOS):

| Web Project | Type | Tabs |
|---|---|---|
| **ADY** | accounting | Dashboard, Budget, Transactions, Reports, Achievements, Documents, Settings, Launchpad, Finance, Charts, Project Management |
| **Main Website** | website | Dashboard, Settings, Website, + all admin tabs |
| **Riad Shannak** | real_estate | Dashboard, Buildings, Achievements, Documents, Reports, Settings, Launchpad |

**iOS has:**
| iOS Project | Tabs |
|---|---|
| ADY Project | Dashboard, Budget, Transactions, Reports, Achievements, Documents |
| Tools & Website | Submissions, LaTeX, Carousel, CRM, Careers, Messages |
| Riad Shannak | Buildings, Documents |

**Differences to discuss:**
- Web ADY has more tabs (Finance, Charts, Project Management, Launchpad) — should iOS add these?
- iOS Tools & Website has Careers and Messages — web has these too but under different tabs
- Web has Launchpad (visual overview of all pages) — iOS doesn't have this. Should it?

---

## 5. Tab projectTypes Updated for real_estate

Added `real_estate` to these tabs so Riad Shannak gets them:
- `dashboard` (was missing)
- `documents` (was missing)
- `reports` (was missing)
- `admin-settings` (was missing)
- `launchpad` (was missing)

Previously only `buildings` and `achievements` had `real_estate` — so Riad Shannak only showed 2 tabs.

---

## 6. Your Design System — Applied ✅

We read `IOS_DESIGN_SYSTEM.md` and have already aligned:
- ✅ Flat cards (border-only, no shadow) — `bg-white border border-gray-100 rounded-lg`
- ✅ Card-based lists instead of tables
- ✅ Status pills (capsule, 14% opacity bg, full-saturation text)
- ✅ Primary button (gray-900, 50px height, subtle shadow)
- ✅ Input fields (gray-50 bg, gray-100 border, 8px radius)
- ✅ Emerald accent for success/active
- ✅ Cairo + Poppins fonts
- ✅ Section eyebrows (uppercase, 11px semibold, gray-400)

---

## 7. Questions for iOS

1. **ADY achievements:** You're now reading from `timeline_entries` right? (We posted `WEB_ACHIEVEMENTS_FIX.md` about this earlier — please confirm you've updated)

2. **Riad Shannak tabs:** We added dashboard, documents, reports, settings, and launchpad to Riad Shannak. Your iOS only has Buildings + Documents. Do you want to add more tabs to Riad Shannak on iOS?

3. **Launchpad:** We have a Launchpad page (visual grid of all available pages with category headers and quick stats). Do you want something similar on iOS? It could be the first screen after picking a project.

4. **Project switching:** We added a project switcher dropdown in the Launchpad hero. On iOS you use an "Exit" button to return to the project hub. Same concept, different UX — both work.

5. **Your IOS_NEW_TABS.md:** We read it. You added Documents, Contact Messages, and Careers. We already have Documents and Careers on web. Contact Messages — we have `contact_submissions` collection. Should we add a Contact Messages tab to the web portal too?

6. **Sub-collections for timeline_entries:** The old system had `timeline_entry_fields`, `timeline_tags`, `timeline_links` sub-collections. We haven't restored these. Do you need them? Or keep it simple with just the main `timeline_entries` collection?

---

## 8. What's Next on Web

- Add Contact Messages tab (if you confirm)
- Review your `IOS_DATABASE_AUDIT.md` for any collections we're missing
- Consider adding Finance/Charts tabs to iOS ADY (or tell us if you don't need them)
- Build CV Center admin view (you asked about our AI extraction approach — we'll share soon)

---

All changes deployed to production: https://robotics-website-5593f.web.app
All pushed to git. Please pull `shared/` for latest docs.
