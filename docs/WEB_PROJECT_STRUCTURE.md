# Web → iOS: Project Structure + Full Page List (July 2)

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Aligning project structure + sharing our full page list so we can plan together

---

## 1. Current Web Projects

Right now the web portal auto-creates 2 projects:
1. **ADY Financial** (type: `accounting`) — financial planning, transactions, budgets
2. **Main Website** (type: `website`) — job submissions, content, contact messages

We're missing **Riad Shannak**. We want to match your 3-project structure:

| Project | iOS Tabs | Proposed Web Tabs |
|---|---|---|
| **ADY Project** | Dashboard, Budget, Transactions, Reports, Achievements | Dashboard, Budget, Transactions, Reports, Charts, Documents, Finance, Operations |
| **Tools & Website** | Submissions, LaTeX, Carousel, CRM | Submissions, LaTeX Compiler, Website, Contact Messages, Careers, Video Tutorials, Video Assignments, CV Preparation, Invitations |
| **Riad Shannak** | Buildings | Dashboard, Buildings (new), Documents, Reports |

**Question for iOS:** What data does Riad Shannak have? We need to know:
- What collections store building/rent data?
- What fields do building documents have?
- Do you have tenants, units, rent payments?
- Should we create a `buildings` collection or do you already have one?

---

## 2. Full Web Page List (28 tabs)

The web portal has **28 registered tabs** organized by category. Here's the complete list:

### Overview (5 tabs)
| Tab ID | Label | Purpose |
|---|---|---|
| `launchpad` | Launchpad | Visual home page with all pages as cards |
| `dashboard` | Dashboard | Project overview with widgets, stats, recent transactions |
| `documents` | Documents | Project documents / evidence files |
| `student-dashboard` | Student Dashboard | Student's personal view (courses, CVs, videos) |
| `investor-portal` | Investor Portal | External client/investor overview |

### Finance (4 tabs)
| Tab ID | Label | Purpose |
|---|---|---|
| `finance` | Finance | Widget-grid overview (revenue, costs, net) |
| `transactions` | Transactions | Full transaction management with card-based list |
| `budget` | Budget | Budget periods + installments |
| `reports` | Reports & Analytics | Financial reports, charts, export |

### Student (6 tabs)
| Tab ID | Label | Purpose |
|---|---|---|
| `cv-preparation` | CV Center | Admin CV review + AI extraction + LaTeX generation |
| `cv-builder` | CV Builder | Student-facing CV builder |
| `student-overview` | Student Management | Manage all students, view their portal |
| `course-management` | Courses | Course catalog management |
| `membership-management` | Memberships | Membership tiers and subscriptions |
| `video-assignment` | Video Assignments | Assign videos to students |

### Operations (2 tabs)
| Tab ID | Label | Purpose |
|---|---|---|
| `operations` | Operations | Placeholder for future operational tools |
| `latex-compiler` | LaTeX Compiler | Compile LaTeX CVs, AI feedback |

### Website (5 tabs)
| Tab ID | Label | Purpose |
|---|---|---|
| `website` | Website | Public website content management |
| `submissions` | Submissions | Job application / CV submissions |
| `contact_messages` | Messages | Contact form submissions |
| `careers-management` | Careers | Job posting management |
| `video-management` | Video Tutorials | Video tutorial catalog |

### Administration (6 tabs)
| Tab ID | Label | Purpose |
|---|---|---|
| `projects` | Projects | Create/edit/delete portal projects |
| `user-management` | User Management | Manage users, roles, admin tools |
| `role-management` | Roles | Role permissions configuration |
| `admin-settings` | Admin | System settings |
| `agent-log` | AI Log | AI assistant conversation log |
| `testing-lab` | Testing Lab | Feature testing sandbox |

---

## 3. iOS vs Web Comparison

| Feature | iOS | Web |
|---|---|---|
| Projects | 3 (ADY, Tools, Riad Shannak) | 2 (ADY, Website) — **need to add Riad Shannak** |
| Auth | Firebase Auth | Firebase Auth ✅ aligned |
| Navigation | Bottom tab bar | Sidebar (desktop-appropriate) |
| Data display | Card-based lists | Card-based lists ✅ aligned |
| Design | Flat cards, emerald accent | Flat cards, emerald accent ✅ aligned |
| Email | Calling our Cloud Functions | Cloud Functions ✅ aligned |
| Pages | ~12 tabs across 3 projects | 28 tabs across 2 projects |
| Extra web-only | — | Launchpad, Student Portal, CV Center, LaTeX, Testing Lab, AI Log, Role Management, Investor Portal, Charts, Operations |

---

## 4. What We Need from iOS

1. **Riad Shannak data structure:** What collections/fields do you use for buildings? We'll create the same structure on web.

2. **Achievements tab:** You have an "Achievements" tab for ADY. What data does it show? We don't have this — should we add it?

3. **CRM tab:** You have a CRM tab under Tools & Website. We have `contact_messages` but no proper CRM. What fields/collections does your CRM use?

4. **Carousel tab:** You mentioned a Carousel tab. Is this a website image carousel? We have website content management but no specific carousel editor.

5. **Which of our extra tabs should iOS add?** We have 16 tabs you don't have. Which ones are useful for iOS? For example:
   - **CV Center** (admin CV review + AI) — could be useful on iOS
   - **LaTeX Compiler** — probably web-only (needs server)
   - **Student Management** — could be useful on iOS
   - **Role Management** — probably admin-only on web
   - **Testing Lab** — web-only
   - **AI Log** — web-only

6. **Project type mapping:** Our project types are: `accounting`, `education`, `client_interface`, `consulting`, `internal`, `website`. What types do you use? Should we add a `real_estate` type for Riad Shannak?

---

## 5. Proposed Next Steps

1. We'll add **Riad Shannak** as a third project (type: `internal` or new type `real_estate`)
2. We'll create a **Buildings** tab if you share the data structure
3. We'll align project names and tabs to match your 3-project model
4. Together we decide which web-only tabs iOS should adopt
5. Together we decide which iOS-only tabs web should add (Achievements, CRM, Carousel)

Let's figure this out together — reply with your data structures and thoughts.
