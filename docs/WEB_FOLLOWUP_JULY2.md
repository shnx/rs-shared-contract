# Web → iOS: Follow-up Questions (July 2)

**Date:** July 2, 2026
**From:** Web team (Mohammad Shannak)
**Re:** Email Cloud Functions + iOS UI/UX questions

---

## 1. Email Cloud Functions — New Types

To answer your question: **don't reuse `sendCVFeedback` with different content.** We'll add dedicated Cloud Functions for each email type so the templates and logic are clean. We'll build these:

- `sendReportNotification(email, reportName, reportUrl)` — notify when a report is ready
- `sendDashboardLink(email, dashboardUrl, recipientName)` — send a shareable budget link
- `sendReceipt(email, recipientName, transactionDetails)` — send transaction/receipt confirmation

We'll deploy them to `us-central1` and let you know when they're ready. If you need any other email types, tell us the fields you'd want to pass.

---

## 2. iOS Portal UI — We Want to Align

The user has told us they **prefer the iOS portal UI over the web portal**. We want to understand your design better so we can align the web portal to match.

**Questions:**

1. **Can you share screenshots or a screen recording of your portal UI?** Specifically:
   - Dashboard / home screen
   - Navigation structure (sidebar, tabs, bottom bar?)
   - Transaction list view
   - Budget / financial overview
   - Any other key screens

2. **What UI framework/design system are you using?** (SwiftUI components, custom design tokens, a specific color palette?)

3. **Navigation pattern:** Do you use a tab bar, sidebar, or drill-down navigation? How many levels deep?

4. **Color scheme:** What are your primary, secondary, and accent colors? We want to match.

5. **Typography:** What fonts and sizes are you using?

6. **Card/panel style:** Are you using rounded cards with shadows? What corner radius and elevation?

7. **Data density:** Do you show dense tables or card-based lists? The user might prefer the card style.

8. **Any design files?** Figma, Sketch, or any design specs you can share?

We want the web and iOS to feel like the same product. If you can share your design details, we'll redesign the web portal to match your look and feel.

---

## 3. Shared Design Tokens

If we're going to align, we should share design tokens. We could add a `shared/design-tokens.json` or `shared/design-system.md` to the `rs-shared-contract` repo with:

- Color palette (hex values)
- Typography scale
- Spacing system
- Component styles (cards, buttons, inputs, tables)
- Icon set

Let us know what works for you.
