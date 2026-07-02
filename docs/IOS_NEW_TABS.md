# iOS → Web: New Tabs Added (July 2)

**Date:** July 2, 2026
**From:** iOS team
**Re:** We've added the 3 tabs you requested

---

## Done ✅

We've added the 3 tabs from your action list:

### 1. Documents Tab ✅
- Reads from `documents` / `ady_documents` (using existing `DocumentsRepository`)
- Shows file name, category, upload date, file type icon (PDF/image)
- Card-based list, sorted by upload date (newest first)
- Available in **ADY Project** and **Riad Shannak**

### 2. Contact Messages Tab ✅
- Reads from `contactInquiries` (using existing `ContactInquiriesRepository`)
- Shows sender name, email, subject, message preview, company, date
- Searchable (by name, email, subject, body)
- Card-based list, sorted by date (newest first)
- Available in **Tools & Website**

### 3. Careers Tab ✅
- Reads from `careers` collection (new `CareersRepository`)
- Shows job title, type (full-time/part-time/internship), description, active/closed status
- Card-based list, sorted by creation date (newest first)
- Available in **Tools & Website**

---

## Updated iOS Tab Structure

| Project | Tabs |
|---|---|
| **ADY Project** | Dashboard, Budget, Transactions, Reports, Achievements, **Documents** |
| **Tools & Website** | Submissions, LaTeX, Carousel, CRM, **Careers**, **Messages** |
| **Riad Shannak** | Buildings, **Documents** |

Total iOS tabs: **14** (was 11)

---

## What's New in Code

- **New file:** `rs/ProjectTabViews.swift` — contains `DocumentsView`, `CareersView`, `ContactMessagesView`
- **New model:** `Career` in `Models.swift`
- **New repository:** `CareersRepository` in `FirestoreRepository.swift`
- **Updated:** `ProjectTab` enum with `.documents`, `.careers`, `.messages`
- **Updated:** `AppProject` tabs and `ProjectContainerView` wiring

All committed and pushed to `devin-1` branch.

---

## Still To Do (from our database audit)

- CV Center (admin CV review + AI) — need your AI extraction approach
- Student Management — depends on student portal scope decision
- Building data model (units, tenants, rent, maintenance) — design together

Let us know what to tackle next.
