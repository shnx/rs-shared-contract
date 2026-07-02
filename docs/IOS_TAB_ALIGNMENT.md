# iOS → Web: Reply to Project Structure + Tab Alignment (July 2)

**Date:** July 2, 2026
**From:** iOS team
**Re:** Answering all your questions + proposing which tabs to mirror

---

## 1. Riad Shannak — Building Data Structure

We have a `buildings` Firestore collection. Here's our `Building` model:

```
Collection: buildings
Fields:
  - name: String
  - address: String
  - unitsCount: Int
  - occupiedUnitsCount: Int
  - managerUserId: String? (Firebase UID of building manager)
  - createdAt: Date?
  - updatedAt: Date?
```

This is **step 1** — read-only scaffolding. We planned but haven't built yet:
- **Units** subcollection (`buildings/{id}/units`) — unit number, floor, area, monthly rent, tenant ID, lease start/end
- **Tenants** collection — name, phone, email, assigned unit
- **Maintenance tickets** — tenant-submitted, triaged by manager
- **Rent payments** — linked to units, track paid/pending

**Recommendation:** Yes, add a `real_estate` project type. We should design the full building data model together before building. For now, the `buildings` collection exists and is read-only.

---

## 2. Achievements Tab — Data Structure

We read from the **`timeline`** Firestore collection (same as the website's timeline). Fallback to `ady_timeline` if empty.

```
Collection: timeline (or ady_timeline)
Fields:
  - title: String
  - description: String?
  - date: Date? (also tries eventDate, achievedAt, timestamp)
  - imageUrl: String? (URL or Firebase Storage path)
  - category: String?
  - projectId: String?
  - createdAt: Date? (also tries uploadedAt, submittedAt)
```

The iOS view shows a visual timeline with images, dates, titles, and descriptions. It's also the source for report extraction.

**Answer:** Yes, you should add an Achievements tab. You already have the `timeline` collection — just build a UI for it. We can share the view structure if needed.

---

## 3. CRM Tab — Data Structure

Our CRM uses the `ady_crm` Firestore collection (prefixed with `ady_`):

```
Collection: ady_crm
Fields:
  - name: String
  - email: String?
  - phone: String?
  - source: String?
  - status: String (enum: new, interested, qualified, demo-promised, active, won, renewal-needed, rejected, lost)
  - notes: String?
  - ownerUserId: String? (Firebase UID of the CRM owner)
  - company: String?
  - value: Double?
  - createdAt: Date?
  - updatedAt: Date?
```

**Your `contact_messages` is different** — that's for contact form submissions from the website. Our CRM is a proper lead pipeline with stages. You should add a CRM tab with this structure.

---

## 4. Carousel Tab

The Carousel is a **content creation tool** — it generates social media carousel slides (like Instagram/LinkedIn carousels) for the website. It's not a website content manager.

```
CarouselSlide:
  - id: UUID
  - template: enum (cover, content, quote, stats, cta, closing)
  - title: String
  - subtitle: String
  - body: String
  - language: enum (en, ar)
  - showLogo: Bool
  - colorHex: String (accent color)
  - slideNumber: Int
  - totalSlides: Int
  - isDarkBackground: Bool
  - ctaButtonText: String
  - imageData: Data? (embedded image)
  - logoPosition: enum (top, bottom)
```

Currently stored locally (exported as images). **This is iOS-only for now** — it's a design tool. You don't need to mirror this on web unless you want a web-based carousel generator.

---

## 5. All Our Firestore Collections

Here's the complete list of collections we use:

| Collection | Project | Purpose |
|---|---|---|
| `systemUsers` | Shared | User accounts (auth, roles, permissions) |
| `ady_transactions` | ADY | Financial transactions |
| `ady_periods` | ADY | Budget periods |
| `ady_installments` | ADY | Budget installments within periods |
| `ady_stores` | ADY | Store/business entities |
| `ady_subscriptions` | ADY | Subscriptions |
| `ady_contracts` | ADY | Contracts |
| `ady_crm` | ADY | CRM leads |
| `ady_documents` | ADY | Project documents |
| `ady_workSessions` | ADY | Employee time tracking |
| `ady_employeePermissions` | ADY | Per-employee permissions |
| `timeline` / `ady_timeline` | ADY | Achievement timeline |
| `buildings` | Riad Shannak | Building management |
| `sharedViews` | Shared | Shareable dashboard links |
| `jobSubmissions` | Tools | Job applications / CV submissions |
| `contactInquiries` | Tools | Contact form messages |
| `careers` | Tools | Job postings (web-managed) |

---

## 6. Which Web-Only Tabs iOS Should Add

Here's our assessment of your 16 extra tabs:

### Should add to iOS:
| Tab | Why |
|---|---|
| **Documents** | Already have `ady_documents` collection — just need a UI |
| **Student Management** | Useful for admin on mobile — view students, their CVs |
| **CV Center** (admin CV review) | We already show submissions — adding AI extraction + LaTeX generation would be powerful |
| **Careers** | Job posting management — admin should be able to post/edit jobs from iOS |
| **Contact Messages** | We already read `contactInquiries` — should add a proper tab |
| **Membership Management** | If we add student features, memberships are needed |

### Maybe later (not priority):
| Tab | Why |
|---|---|
| **Finance** (widget grid) | We have Dashboard + Transactions — might be redundant |
| **Courses** | Depends on whether we build a student portal |
| **Video Assignments** | Depends on student portal |
| **Video Tutorials** | Could be a simple embedded video list |
| **Investor Portal** | We have shared links — might be enough |
| **Launchpad** | iOS uses project hub instead — different pattern |

### Web-only (skip on iOS):
| Tab | Why |
|---|---|
| **LaTeX Compiler** | Needs server — iOS already calls the web API |
| **Role Management** | Admin-only, better on web's full screen |
| **Admin Settings** | System config, better on web |
| **AI Log** | Debug tool, web-only |
| **Testing Lab** | Dev tool, web-only |
| **Operations** | Placeholder, no content yet |

---

## 7. Proposed Unified Tab Structure

### ADY Project
| Tab | iOS | Web | Status |
|---|---|---|---|
| Dashboard | ✅ | ✅ | Aligned |
| Budget | ✅ | ✅ | Aligned |
| Transactions | ✅ | ✅ | Aligned |
| Reports | ✅ | ✅ | Aligned |
| Achievements | ✅ | ❌ | Web needs to add |
| Documents | ❌ | ✅ | iOS needs to add |
| Finance | ❌ | ✅ | Maybe redundant |
| Operations | ❌ | ✅ | Web-only for now |

### Tools & Website
| Tab | iOS | Web | Status |
|---|---|---|---|
| Submissions | ✅ | ✅ | Aligned |
| LaTeX | ✅ (calls web API) | ✅ | Aligned |
| Carousel | ✅ | ❌ | iOS-only (design tool) |
| CRM | ✅ | ❌ | Web needs to add |
| Contact Messages | ❌ | ✅ | iOS needs to add |
| Careers | ❌ | ✅ | iOS needs to add |
| Website | ❌ | ✅ | Web-only (content mgmt) |
| CV Center | ❌ | ✅ | iOS should add |
| Student Mgmt | ❌ | ✅ | iOS should add |
| Video Tutorials | ❌ | ✅ | Maybe later |
| Video Assignments | ❌ | ✅ | Maybe later |
| Memberships | ❌ | ✅ | Maybe later |

### Riad Shannak
| Tab | iOS | Web | Status |
|---|---|---|---|
| Buildings | ✅ (read-only) | ❌ | Web needs to add |
| Dashboard | ❌ | ❌ | Both need to add |
| Documents | ❌ | ❌ | Both need to add |
| Reports | ❌ | ❌ | Both need to add |

---

## 8. Project Type Mapping

Your types: `accounting`, `education`, `client_interface`, `consulting`, `internal`, `website`

Our mapping:
- **ADY Project** → `accounting` (or `education` since it's PSUT Venture Lab)
- **Tools & Website** → `website`
- **Riad Shannak** → needs new type `real_estate`

**Recommendation:** Add `real_estate` type. ADY could be `accounting` or a dual-type.

---

## 9. Next Steps

1. **You:** Add Riad Shannak project (type: `real_estate`) with Buildings tab
2. **You:** Add Achievements tab (read from `timeline` collection)
3. **You:** Add CRM tab (read from `ady_crm` collection)
4. **iOS:** Add Documents tab for ADY (read from `ady_documents`)
5. **iOS:** Add Contact Messages tab (read from `contactInquiries`)
6. **iOS:** Add Careers tab (read from `careers`)
7. **Both:** Design full building data model (units, tenants, rent, maintenance)
8. **Both:** Decide on student portal scope

Let's prioritize getting the existing collections aligned first, then build new features together.
