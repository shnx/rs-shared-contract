# iOS Design System — For Web Team Alignment

**Date:** July 2, 2026
**From:** iOS team
**Re:** Sharing our design system so web can align

Screenshots aren't needed — here's the full design spec extracted from our code.

---

## 1. Color Palette

Based on **Tailwind CSS** grayscale + emerald accent. Already matches the-rs.com.

### Surfaces
| Token | Hex | Tailwind | Usage |
|---|---|---|---|
| `primary` | `#111827` | gray-900 | CTA buttons, primary actions |
| `background` | `#FFFFFF` | white | Card backgrounds, main background |
| `secondaryBackground` | `#F9FAFB` | gray-50 | Secondary surfaces, input fields |
| `tertiary` | `#F3F4F6` | gray-100 | Neutral status pill bg |

### Brand Accent (Emerald)
| Token | Hex | Tailwind | Usage |
|---|---|---|---|
| `accent` | `#10B981` | emerald-500 | Active status, success |
| `accentLight` | `#34D399` | emerald-400 | Hover states |
| `accentDark` | `#059669` | emerald-600 | Links, active text |

### Text
| Token | Hex | Tailwind | Usage |
|---|---|---|---|
| `textPrimary` | `#111827` | gray-900 | Headlines, primary text |
| `textSecondary` | `#6B7280` | gray-500 | Body text, captions |
| `textTertiary` | `#9CA3AF` | gray-400 | Placeholders, meta info |
| `textOnPrimary` | `#FFFFFF` | white | Text on dark buttons |

### Semantic
| Token | Hex | Tailwind |
|---|---|---|
| `success` | `#10B981` | emerald-500 |
| `warning` | `#F59E0B` | amber-500 |
| `error` | `#EF4444` | red-500 |
| `info` | `#3B82F6` | blue-500 |

### Borders
| Token | Hex | Tailwind |
|---|---|---|
| `border` | `#F3F4F6` | gray-100 |
| `divider` | `#E5E7EB` | gray-200 |

### Status Colors (CRM/pipeline)
| Token | Hex | Tailwind |
|---|---|---|
| `statusNew` | `#3B82F6` | blue-500 |
| `statusQualified` | `#8B5CF6` | violet-500 |
| `statusActive` | `#10B981` | emerald-500 |
| `statusWon` | `#10B981` | emerald-500 |
| `statusLost` | `#EF4444` | red-500 |

---

## 2. Typography

**Fonts:** Cairo (Arabic) + Poppins (English/Latin). Falls back to SF Pro / SF Arabic.

### Type Scale
| Role | Size | Weight | Usage |
|---|---|---|---|
| Large Title | 34 | bold | Screen titles |
| Title | 28 | bold | Section headers |
| Title 2 | 22 | bold | Card headers |
| Title 3 | 20 | semibold | Sub-headers |
| Headline | 17 | semibold | Button text, row titles |
| Body | 17 | regular | Default body text |
| Callout | 16 | regular | Secondary body |
| Subheadline | 15 | regular | Metadata, descriptions |
| Footnote | 13 | regular | Small labels |
| Caption | 12 | regular | Status pills, timestamps |

---

## 3. Spacing & Layout

| Token | Value | Usage |
|---|---|---|
| `paddingSmall` | 8px | Tight spacing, pill padding |
| `paddingMedium` | 16px | Card padding, row spacing |
| `paddingLarge` | 24px | Section spacing |
| `paddingXLarge` | 32px | Screen edge padding |

### Corner Radius
| Token | Value | Tailwind | Usage |
|---|---|---|---|
| `cornerRadiusSmall` | 6px | rounded-md | Inputs, small buttons |
| `cornerRadiusMedium` | 8px | rounded-lg | Cards, buttons |
| `cornerRadiusLarge` | 12px | rounded-xl | Large cards, sheets |

### Shadows
- `shadowRadius`: 4px (subtle, used sparingly)
- Cards use **no shadow** — just 1px hairline border (`gray-100`)
- Buttons: `shadow(color: .black.opacity(0.1), radius: 4, y: 2)` — subtle drop
- Login card: `shadow(color: .black.opacity(0.05), radius: 12, y: 4)` — larger soft shadow

---

## 4. Navigation Pattern

**Bottom tab bar** per project. The app has 3 projects, each with its own set of tabs:

| Project | Tabs |
|---|---|
| **ADY Project** | Dashboard, Budget, Transactions, Reports, Achievements |
| **Tools & Website** | Submissions, LaTeX, Carousel, CRM |
| **Riad Shannak** | Buildings |

- User selects a project from a **project hub** (grid of project cards)
- Each project opens a **bottom TabView** with its tabs
- An **Exit** button is always present to return to the project hub
- Navigation within tabs uses **drill-down** (NavigationStack push), typically 2-3 levels deep

---

## 5. Component Styles

### Card
- White background (`#FFFFFF`)
- 1px border (`#F3F4F6`, gray-100)
- 8px corner radius
- 16px internal padding
- **No shadow** (flat design, hairline border only)

### Status Pill
- Capsule shape
- 14% opacity background tinted by status color
- Full-saturation text color (e.g. emerald-600 for active)
- Font: 12px regular
- Padding: 6px horizontal, 4px vertical

### Section Eyebrow
- Uppercased label text
- 11px semibold
- `textTertiary` color (gray-400)
- Letter spacing slightly increased

### Buttons
- **Primary**: gray-900 bg, white text, 8px radius, 50px height, subtle shadow
- **Secondary**: gray-50 bg, gray-900 text, 1px gray-100 border, 8px radius
- **Icon buttons**: SF Symbols, 12-20px, semibold weight

### Input Fields
- gray-50 background
- 1px gray-100 border
- 8px corner radius
- 14px padding
- Left-aligned icon + text in HStack
- 17px body font

---

## 6. Data Density

**Card-based lists**, not dense tables. Each item is a card with:
- Avatar/initials circle (left)
- Title + subtitle (center, stacked)
- Status pill or timestamp (right)

Detail views use stacked cards vertically with 12px spacing between them.

---

## 7. Icons

**Apple SF Symbols** throughout. Key icons:
- Dashboard: `chart.pie.fill`
- Budget: `calendar`
- Transactions: `list.bullet.rectangle`
- Reports: `doc.text.fill`
- Achievements: `trophy.fill`
- Submissions: `tray.full.fill`
- CRM: `person.2.fill`
- Buildings: `building.2.fill`
- Settings: `gearshape.fill`

---

## 8. Language

- **Bilingual**: English + Arabic
- Arabic uses RTL layout
- Locale: `ar_JO` for numbers/dates
- Currency: JOD (Jordanian Dinar)

---

## Summary for Web Team

Our design is essentially **Tailwind grayscale + emerald accent** — which the-rs.com already uses. The key differences from a typical web portal:

1. **Flat cards** (no shadows, just hairline borders)
2. **Card-based lists** instead of dense tables
3. **Bottom tab navigation** (not sidebar)
4. **Large touch targets** (50px buttons, 44px min row height)
5. **Minimal color** — mostly grayscale with emerald for positive actions

You can replicate this on web with your existing Tailwind setup. The main change would be switching from shadow-based cards to border-only cards, and using card-based list layouts instead of tables.
