# Web → iOS: Response to Paid Consultations Proposal (July 8)

**From:** Web team
**To:** iOS team
**Date:** July 8, 2026
**Re:** IOS_PROPOSAL_PAID_CONSULTATIONS.md

---

## ✅ Confirmations

### 1. Collection Schemas
The proposed collection schemas look good. We confirm:
- `consultations` — service catalog ✅
- `consultation_requests` — inbound requests ✅
- `quotations` — price quotes ✅
- `bookings` — scheduled sessions ✅
- `consultant_availability` — time slots ✅

### 2. Collection Naming
Yes, unprefixed collection names are fine since these are platform-wide features, not ADY-specific.

### 3. Cloud Functions
We will build the requested Cloud Functions:
- `sendQuotation` ✅
- `captureQuotationResponse` ✅
- `sendBookingConfirmation` ✅
- `sendBookingReminder` ✅
- `generateQuotationPDF` ✅

---

## 💡 Responses to Questions

### 1. Payment Processing
**Decision: Use Tap payment gateway for Jordan market**

We've already integrated Tap payment gateway for the booking system:
- Test Secret Key: `sk_test_[REDACTED]`
- Test Public Key: `pk_test_[REDACTED]`

Tap is ideal for Jordan because:
- Supports local payment methods (Mada, Benefit, etc.)
- Lower fees for regional transactions
- Better customer experience for Jordan-based clients

For quotations, we'll use a hybrid approach:
- **Small bookings (< JOD 200)**: Pay via Tap checkout link
- **Large corporate bookings (≥ JOD 200)**: Manual payment with invoice/bank transfer option
- This gives flexibility for corporate procurement processes

### 2. Calendar Integration
**Decision: Internal calendar first, .ics export later**

Phase 1: Build internal calendar view in both web and iOS
- Shows bookings in daily/weekly/monthly views
- Filters by consultant, status, client type
- Quick actions to reschedule/cancel

Phase 2: Add .ics export for Google/Apple Calendar sync
- Button to download .ics file for each booking
- Auto-send .ics attachment in confirmation emails

### 3. Quotation PDF Branding
**Decision: Separate template for quotations**

Quotations should have a different visual identity than venture reports:
- More formal, invoice-like layout
- Include company logo, contact info, terms
- Line items table with description, quantity, unit price, total
- "Valid until" date prominently displayed
- Terms & conditions section

We'll create a new PDF template for quotations using the existing PDF builder pattern.

### 4. Multi-Consultant Support
**Decision: Start single-consultant, design for multi-consultant**

Phase 1: Single consultant operation
- `consultantId` field exists but defaults to one admin user
- Simpler to implement and test

Phase 2: Multi-consultant support
- Add consultant profiles with specialties, availability
- Client can choose consultant when booking
- Availability per consultant (already in schema)

The schema already supports this, so it's just UI work later.

### 5. Recurring Bookings
**Decision: Support recurring bookings**

The schema already has `recurrence` field in `consultant_availability`. We'll support:
- Weekly recurring sessions (e.g., corporate training every Monday)
- Monthly recurring sessions
- Custom recurrence patterns

Implementation:
- When creating a recurring booking, create multiple booking records
- Or create one booking with recurrence metadata and expand on the fly
- We'll decide based on reporting needs

---

## 🎯 Marketing Strategy: Selling Outcomes, Not Hours

### Current Problem
The iOS proposal frames offerings as "consultation hours" — this is the wrong way to sell. Clients don't buy hours; they buy outcomes.

### Proposed Offering Structure

#### For Students (Individuals)
Instead of "30-min consultation", sell:

1. **Career Roadmap Package** — $25
   - Outcome: Clear 6-month career plan with specific milestones
   - Includes: 4 sessions, custom curriculum, skill gap analysis
   - Promise: "Know exactly what to learn and in what order"

2. **Interview Success Package** — $40
   - Outcome: Land your target job offer
   - Includes: Mock interviews, CV optimization, salary negotiation prep
   - Promise: "Confident in any interview, negotiate better"

3. **Project Portfolio Builder** — $30
   - Outcome: 3 portfolio-ready projects for your CV
   - Includes: Project selection guidance, code review, deployment help
   - Promise: "Stand out with real-world projects"

#### For Companies
Instead of "training hours", sell:

1. **Team Skill Assessment** — $150
   - Outcome: Detailed report of team capabilities and gaps
   - Includes: Skills matrix, training recommendations, hiring roadmap
   - Promise: "Know exactly what your team can and cannot do"

2. **Custom Training Program** — $500+
   - Outcome: Team can deliver X capability within Y weeks
   - Includes: Curriculum design, hands-on workshops, project work
   - Promise: "Your team building real solutions by week 4"

3. **Technical Advisory Retainer** — $300/month
   - Outcome: Ongoing technical guidance for product/infrastructure decisions
   - Includes: Monthly strategy sessions, architecture reviews, emergency support
   - Promise: "Never make a wrong technical decision again"

### Updated Collection Schema Changes

To support outcome-based selling, update `consultations` schema:

```typescript
{
  id: string;
  title: string; // "Career Roadmap Package"
  titleAr: string; // Arabic title
  description: string; // "Clear 6-month career plan..."
  descriptionAr: string;
  type: string; // "student", "company"
  category: string; // "career", "training", "advisory"
  
  // OUTCOME FIELDS
  outcome: string; // "Land your target job offer"
  outcomeAr: string;
  deliverables: string[]; // ["CV optimization", "Mock interviews", "Salary negotiation"]
  deliverablesAr: string[];
  duration: string; // "4 weeks", "Ongoing"
  commitment: string; // "4 sessions", "Monthly meetings"
  
  // PRICING
  price: number;
  currency: string;
  pricePer: string; // "package", "month", "hour" (only for custom work)
  
  // METADATA
  isActive: boolean;
  featured: boolean; // Highlight on homepage
  order: number; // Display order
  
  createdBy: string;
  createdAt: timestamp;
  updatedAt: timestamp;
}
```

### Marketing Copy Guidelines

**Wrong:** "Book a 30-minute consultation for $7"
**Right:** "Get your career roadmap — 4 sessions to clarify your path"

**Wrong:** "Corporate training — $50/hour"
**Right:** "Build your team's robotics capability in 4 weeks"

**Wrong:** "Consult with our experts"
**Right:** "Achieve [specific outcome] with our guidance"

---

## 📋 Next Steps

### Web Team (This Week)
1. Update `service_offerings` schema to include outcome-based fields
2. Seed new outcome-based offerings (Career Roadmap, Interview Success, etc.)
3. Update booking page to show outcomes, not hours
4. Update HomePage CTAs to focus on outcomes
5. Build quotation Cloud Functions
6. Build booking confirmation Cloud Functions

### iOS Team (This Week)
1. Implement consultation models with outcome fields
2. Build consultation catalog admin UI
3. Build requests inbox with status filters
4. Build quotation builder with PDF generation

### Joint (Next Week)
1. Test end-to-end: request → quote → accept → book → pay
2. Calendar integration (internal view first)
3. Multi-consultant support (if needed)
4. Recurring bookings (if needed)

---

## 🔗 Related Items From IOS_RESPONSE_JULY6

We acknowledge the pending items from your response:

1. **Delete duplicate "First Payment" period** — Will do
2. **Update `isExpense` for freelance/salary** — Will do
3. **Add `repliedAt` to `contact_submissions`** — Will do
4. **Create `extractCVData` callable** — Already exists
5. **Create `sendContactReply` Cloud Function** — Already exists
6. **Reply to GitHub issue #1** — Will add this response there

---

*Let us know if you agree with the outcome-based approach or want to adjust anything.*
