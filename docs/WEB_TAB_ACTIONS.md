# Web → iOS: Actions Taken on Tab Alignment (July 2)

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** We've implemented your proposals — here's what's done

---

## What We Built

### 1. Riad Shannak Project ✅
- Added `real_estate` project type
- Created Riad Shannak project in database initialization
- Default tabs: Dashboard, Buildings, Documents, Reports, Settings
- Reads from `buildings` Firestore collection (your exact schema)

### 2. Buildings Tab ✅
- New page at `src/pages/Buildings.tsx`
- Card-based list with avatar, name, address, occupancy stats
- Full CRUD: add, edit, delete buildings
- Shows unit count + occupied count + occupancy percentage
- iOS design: flat cards, status pills, no shadows

### 3. Achievements Tab ✅
- New page at `src/pages/Achievements.tsx`
- Reads from `timeline` Firestore collection (with fallbacks: `eventDate`, `achievedAt`, `timestamp`)
- Visual timeline with dots, dates, images, categories
- Full CRUD: add, edit, delete achievements
- Available for ADY + Riad Shannak projects

### 4. CRM Tab ✅
- New page at `src/pages/CRM.tsx`
- Reads from `ady_crm` Firestore collection (your exact schema)
- Lead pipeline with all 9 statuses: new, interested, qualified, demo-promised, active, won, renewal-needed, rejected, lost
- Status filter pills with counts
- Inline status change via dropdown
- Full CRUD: add, edit, delete leads
- Shows: name, company, email, phone, value (JOD), status

### 5. Renamed "Main Website" → "Tools & Website" ✅
- Now matches your project name
- Updated default tabs to include: submissions, latex-compiler, website, contact_messages, careers-management, video-management, cv-preparation, invitation-manager, crm

---

## Updated Project Structure

| Project | Type | Default Tabs |
|---|---|---|
| **ADY Financial** | `accounting` | dashboard, budget, transactions, reports, achievements, documents, finance, operations, settings |
| **Tools & Website** | `website` | submissions, latex-compiler, website, contact_messages, careers-management, video-management, cv-preparation, invitation-manager, crm, settings |
| **Riad Shannak** | `real_estate` | dashboard, buildings, documents, reports, achievements, settings |

---

## Total Tabs: 31 (was 28)

Added 3 new tabs:
- `buildings` (Operations, real_estate only)
- `achievements` (Overview, accounting + real_estate)
- `crm` (Operations, accounting + website + consulting + client_interface)

---

## What iOS Should Do Next (per your proposal)

1. **Add Documents tab** — read from `ady_documents`
2. **Add Contact Messages tab** — read from `contactInquiries`
3. **Add Careers tab** — read from `careers`
4. **Add CV Center** — we can share our AI extraction + LaTeX generation approach

## What We Should Design Together

1. Full building data model (units, tenants, rent, maintenance)
2. Student portal scope
3. Whether ADY should be `accounting` or `education` (or dual-type)

---

## Firestore Collections We Now Share

| Collection | Web reads? | Web writes? | iOS reads? | iOS writes? |
|---|---|---|---|---|
| `buildings` | ✅ | ✅ | ✅ | ✅ |
| `ady_crm` | ✅ | ✅ | ✅ | ✅ |
| `timeline` | ✅ | ✅ | ✅ | ✅ |
| `ady_transactions` | ✅ | ✅ | ✅ | ✅ |
| `ady_periods` | ✅ | ✅ | ✅ | ✅ |
| `ady_documents` | ✅ | ✅ | ❌ (needs to add) | ❌ |
| `jobSubmissions` | ✅ | ✅ | ✅ | ✅ |
| `contactInquiries` | ✅ | ✅ | ✅ | ✅ |
| `careers` | ✅ | ✅ | ❌ (needs to add) | ❌ |
| `systemUsers` | ✅ | ✅ | ✅ | ✅ |

Let us know if anything needs adjusting.
