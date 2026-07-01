# Category Taxonomy

The iOS app uses a **two-tier category system**: 6 broad categories, each with sub-categories. The web should replicate this for consistency.

## Broad Categories

| Broad Category | Icon (SF Symbol) | Color (hex) | Sub-Categories |
| --- | --- | --- | --- |
| Hardware | `cpu` | `#3B82F6` | Components, Prototyping |
| Software | `laptopcomputer` | `#8B5CF6` | Subscriptions, Infrastructure |
| Personnel | `person.2.fill` | `#6366F1` | Internship, Salaries, Development |
| Operations | `gearshape.fill` | `#10B981` | Marketing, Travel, Food, Office |
| Fees | `creditcard.fill` | `#64748B` | Bank Fees, Card Payments |
| Funding | `banknote.fill` | `#059669` | Grants, Installments |
| Other (fallback) | `tag.fill` | `#9CA3AF` | — |

## Keyword Matching

Each broad category and sub-category has keywords. When a transaction's `description`, `vendor`, `category`, or `note` contains a keyword (case-insensitive), it's classified into that category.

### Hardware
- **Components:** robot, motor, arduino, raspberry, sensor, servo, circuit, electronic, component, wire, board, battery
- **Prototyping:** 3d print, filament, metal, prototype, parts, hardware

### Software
- **Subscriptions:** license, subscription, saas, pro plan, plan, tool, stackblitz, github, openai, claude, ai model, figma, notion, subscription time
- **Infrastructure:** hosting, domain, cloud, api, software

### Personnel
- **Internship:** internship, intern, trainee
- **Salaries:** salary, wage, payroll, stipend
- **Development:** development, coding, programming, dev, engineering

### Operations
- **Marketing:** ads, advert, marketing, promo, branding, campaign
- **Travel:** taxi, uber, careem, flight, hotel, fuel, petrol, transport, mileage
- **Food:** restaurant, cafe, coffee, lunch, meal, food, catering, snack
- **Office:** paper, pen, stationery, printer, ink, furniture, desk, supplies

### Fees
- **Bank Fees:** fee, bank, commission, charge
- **Card Payments:** mastercard, visa

### Funding
- **Grants:** fund, grant, disbursement
- **Installments:** installment, payment received

## Storage Format

- Broad only: `"Hardware"`
- Broad + sub: `"Hardware > Components"`
- Legacy/free text: stored as-is (e.g. `"Arduino sensors"`)
- Missing: falls back to transaction type display name (e.g. `"Cost"`)

## Consolidation

Map any free-text category to a broad category by keyword matching:
1. Check if the text matches a known broad category name (case-insensitive)
2. If not, run keyword matching against all broad category keywords
3. If no match → "Other"

This is used for the "Spending by Category" card and "Top Categories" in spending insights.
