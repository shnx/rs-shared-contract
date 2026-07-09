# Web → iOS: Project Consolidation — Exactly 3 Projects

**From:** Web team
**To:** iOS team
**Date:** July 9, 2026

---

## What We Did

We consolidated all projects (there were 6-7) into exactly **3 canonical projects**. No more, no less.

### The 3 Projects

| # | Name | Type | Firestore `type` field | Purpose |
|---|------|------|------------------------|---------|
| 1 | **ADY** | `accounting` | `"accounting"` | Financial planning, receipts, transactions, periods, stores, accounting |
| 2 | **Riad Shannak** | `real_estate` | `"real_estate"` | Real estate: buildings, units, tenants, rent tracking |
| 3 | **Website and Tools** | `website` | `"website"` | Job submissions, CV prep, courses, bookings, consultations, proposals, CRM, website content |

### Arabic Names
- ADY → `آدي`
- Riad Shannak → `رياض شناك`
- Website and Tools → `الموقع والأدوات`

---

## How We Did It

### 1. Seed/Migration Logic (`initializeDatabase`)

On app load, we now:

1. Fetch all existing projects from Firestore `projects` collection
2. For each of the 3 canonical types (`accounting`, `real_estate`, `website`):
   - If a project with that type **exists** → rename it to the canonical name (e.g. "ADY Financial" → "ADY")
   - If it **doesn't exist** → create it with the canonical name and default tabs
3. Old projects (type `internal`, `education`, `consulting`, `client_interface`) are **not deleted** but are **hidden** from the project selector

### 2. Project Filtering (`ProjectContext`)

The `refreshProjects()` function now filters to only show projects whose `name` matches one of:

```
['ADY', 'Riad Shannak', 'Website and Tools']
```

Case-insensitive match. All other projects in Firestore are invisible in the portal.

### 3. Tab Project Types — Consolidated

All tabs previously used 7 different `projectTypes` values:
`accounting`, `education`, `client_interface`, `consulting`, `internal`, `website`, `real_estate`

We replaced them with only 3:

| Old type | New type | Reason |
|----------|----------|--------|
| `internal` | `accounting` + `real_estate` + `website` | Admin/settings tabs should show in all projects |
| `education` | `website` | Student/course tabs belong in Website and Tools |
| `consulting` | `website` | Booking/consultation tabs belong in Website and Tools |
| `client_interface` | `website` | Client-facing tabs belong in Website and Tools |

### 4. Which Tabs Show in Which Project

**ADY (`accounting`):**
- Dashboard, Documents, Timeline, Finance, Transactions, Budget, Reports
- Admin tabs: Projects, User Management, Role Management, Settings, Agent Log, Testing Lab

**Riad Shannak (`real_estate`):**
- Dashboard, Documents, Riad Portal, Buildings, Reports
- Admin tabs: same as above

**Website and Tools (`website`):**
- Dashboard, Documents, Submissions, Student Portal, Student Overview
- Course Management, Membership Management, CV Preparation, Video Hub, LaTeX Compiler
- CRM Contacts, Email Inbox, Service Offerings, Proposal Generator
- Availability, Bookings, Tap Config, Website Manager
- Admin tabs: same as above

---

## What iOS Should Do

1. **Use the same 3 project names** when creating/displaying projects
2. **Use the same `type` values**: `"accounting"`, `"real_estate"`, `"website"`
3. **Stop creating projects** with types: `internal`, `education`, `consulting`, `client_interface`
4. **If you have existing projects** with old types, either:
   - Update their `type` field in Firestore to the closest canonical type
   - Or delete them and let the web seed recreate the 3 canonical ones
5. **Filter your project list** the same way: only show projects named `ADY`, `Riad Shannak`, `Website and Tools`

---

## Firestore Impact

- The `projects` collection may still contain old projects — they're just hidden
- No data is deleted or moved
- Transactions, periods, documents etc. are **not affected** — they still reference the same project IDs
- If iOS was writing to an old project ID, that data still exists but won't show in the portal until the project is renamed

## Questions for iOS

1. Do you have any projects in Firestore with types other than `accounting`/`real_estate`/`website`?
2. If yes, should we write a migration script to reassign their data to the canonical project?
3. Are you using the `type` field to filter tabs/screens in the iOS app? If so, you'll need to update those filters to use only the 3 types.
