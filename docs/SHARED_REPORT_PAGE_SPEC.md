# Shared Report Page — Web Spec

> **Goal:** When an iOS user creates a shareable link from the ADY Reports tab, the web should show a public page at `/shared/:token` that looks exactly like the Venture Lab PDF report, but with **clickable receipt buttons** and a **Download as PDF** button.

---

## 1. Firestore Document

The iOS app writes to `sharedViews` collection with this structure:

```
sharedViews/{token}
{
  projectId: string,
  createdBy: string,
  token: string,
  label: string,                    // "ADY"
  createdAt: Timestamp,
  revoked: boolean,                 // if true, show "link expired" page

  reportTemplate: "venture_lab",    // tells web which template to render

  fund: {
    name: string,                   // "PSUT Venture Lab"
    total: number,                  // 10000
    received: number,               // PSUT income total
    spent: number,                  // PSUT expense total
    inCash: number,                 // received - spent (CAN BE NEGATIVE)
    toReceive: number,              // max(0, total - received)
    receivedPct: number             // round(received / total * 100)
  },

  summary: {
    totalIncome: number,            // ALL income (all funding sources)
    totalSpent: number,             // ALL expenses (all funding sources)
    netBalance: number,             // totalIncome - totalSpent
    transactionCount: number,
    periodCount: number,
    isOverspent: boolean,           // totalSpent > totalIncome
    overspendAmount: number,        // max(0, totalSpent - totalIncome)
    generatedAt: number             // epoch ms
  },

  periods: [
    {
      id: string,
      name: string,
      startDate: number,            // epoch ms
      endDate: number,              // epoch ms
      allocatedFunds: number,
      income: number,
      spent: number,
      net: number,
      transactionCount: number
    }
  ],

  carryOverTimeline: [
    {
      periodId: string,
      periodName: string,
      periodNet: number,
      carryIn: number,
      carryOut: number,             // negative = debt (show red)
      isDebt: boolean
    }
  ],

  monthlyBuckets: [
    {
      id: string,                   // "yyyy-MM"
      title: string,                // "July 2026"
      income: number,
      expense: number,
      net: number,
      count: number
    }
  ],

  transactions: [
    {
      id: string,
      date: number,                 // epoch ms
      type: string,                 // "cost" | "bill" | "payment" | "check" | "revenue"
      category: string,
      description: string,          // display title
      amount: number,               // original amount
      amountJOD: number,            // converted to JOD
      currency: string,             // "JOD" or "USD"
      vendor: string,
      fundingSource: string,        // "psut", "self", etc.
      note: string,
      isExpense: boolean,
      isIncome: boolean,
      hasEvidence: boolean,         // true if receipt/bill attached
      evidenceFiles: [
        {
          id: string,
          fileName: string,
          fileType: string,         // "application/pdf", "image/jpeg"
          fileSize: number
        }
      ],
      evidenceCount: number         // total evidence files + linked documents
    }
  ],

  meetings: [
    {
      id: string,
      meetingDate: number,          // epoch ms
      mainPoints: string,
      decisions: string
    }
  ],

  achievements: [
    {
      id: string,
      title: string,
      summary: string,
      date: number,                 // epoch ms
      entryType: string,            // "milestone", "income", "expense", etc.
      imageUrl: string              // Firebase Storage URL
    }
  ]
}
```

---

## 2. Page Layout

### URL: `/shared/:token`

### Check on load:
1. Read `sharedViews/{token}` from Firestore
2. If document doesn't exist → show "Link not found" page
3. If `revoked == true` → show "This link has been expired" page
4. Otherwise → render the report page

### Page Structure (top to bottom):

```
┌─────────────────────────────────────────────┐
│  [Download PDF]  [Share Link]               │  ← sticky header
├─────────────────────────────────────────────┤
│                                             │
│         PSUT Venture Lab                    │
│    Project Progress & Budget Report         │
│                                             │
│  01  Project Information                    │
│  ┌───────────────────────────────────────┐  │
│  │ Project Title:    ADY — ...           │  │
│  │ Project Manager:  Mohammad Shannak    │  │
│  │ Report Date:      3 Jul 2026          │  │
│  │ Reporting Period: ...                 │  │
│  │ Current Stage:    Active              │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  02  Meeting with Sari Awwad                │
│  ┌───────────────────────────────────────┐  │
│  │ Meeting Date:     15 Jun 2026         │  │
│  │ Main Points:      ...                 │  │
│  │ Decisions:        ...                 │  │
│  └───────────────────────────────────────┘  │
│  (repeat for each meeting)                  │
│                                             │
│  03  Achievements & Work Completed          │
│  ┌───────────────────────────────────────┐  │
│  │ • [icon] Title (date)                 │  │
│  │   Summary text...                     │  │
│  │   [image if imageUrl]                 │  │
│  └───────────────────────────────────────┘  │
│  (repeat for each achievement)              │
│                                             │
│  04  Challenges & Risks                     │
│  (static blank fields)                      │
│                                             │
│  05  Support & Assistance Needed            │
│  (static checkboxes)                        │
│                                             │
│  06  Budget & Spending                      │
│  ┌───────────────────────────────────────┐  │
│  │ Total Received:  JD 5,000.00          │  │
│  │ Total Spent:     JD 3,200.00          │  │
│  │ Balance:         JD 1,800.00          │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  Transaction Log — All Transactions         │
│  ┌──────┬──────┬─────────┬────────┬──────┬──┐│
│  │Date  │Type  │Amount   │Category│Reason│📄││
│  ├──────┼──────┼─────────┼────────┼──────┼──┤│
│  │3 Jun │Cost  │JD 50.00 │Equip.  │Cables│📎││ ← clickable
│  │5 Jun │Payment│+JD 2000│Income  │Grant │— ││
│  │7 Jun │Bill  │JD 120.00│Utilities│Electric│📎││ ← clickable
│  └──────┴──────┴─────────┴────────┴──────┴──┘│
│                                             │
│  When user clicks 📎 button:                 │
│  → If evidenceFiles exist, open a modal     │
│    showing the receipt/bill                 │
│  → Modal has: file name, file type, download│
│    button, and inline preview if PDF/image  │
│                                             │
│  Planned Future Spending                    │
│  (table from plannedItems)                  │
│                                             │
│  07  Budget Overview & Summary              │
│  (static blank field)                       │
│                                             │
│  PSUT Grant Card:                           │
│  ┌───────────────────────────────────────┐  │
│  │ PSUT Venture Lab                      │  │
│  │ [progress bar: spent | cash | pending]│  │
│  │ Spent: JD 3,200  In cash: JD 1,800   │  │
│  │ To receive: JD 5,000                  │  │
│  │ (inCash shows RED if negative)        │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  Period Summary Table:                      │
│  ┌────────┬────────┬────────┬──────┬────┬────┐│
│  │Period  │Planned │Received│Spent │Net │Carry││
│  ├────────┼────────┼────────┼──────┼────┼────┤│
│  │Jun 2026│JD 5000 │JD 5000 │JD 3200│+1800│+1800││
│  └────────┴────────┴────────┴──────┴────┴────┘│
│  (totals row at bottom)                     │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 3. Clickable Receipt Buttons

For each transaction in the transaction log table:

- If `hasEvidence == true`, show a **📎 button** (or "View Receipt" link)
- Clicking opens a **modal/overlay** that:
  1. Shows the file name and type
  2. If `fileType` is `application/pdf`: embeds a PDF viewer (`<iframe>` or `<embed>`)
  3. If `fileType` starts with `image/`: shows the image inline
  4. Has a **Download** button to download the original file
  5. Has a **Close** button

### How to serve evidence files:

The evidence files are stored as base64 in the transaction's `evidenceFiles` array in Firestore (`ady_transactions` collection). The web needs to:

1. Read the transaction document from `ady_transactions/{transactionId}`
2. Get `evidenceFiles` array
3. For the clicked file, decode `base64Data` and serve it as a blob

**Alternative (recommended):** When generating the share link, the iOS app could upload evidence files to Firebase Storage and include the download URLs in the `evidenceFiles` array. For now, the web should fetch from Firestore directly.

### Evidence fetching from Firestore:

```javascript
async function fetchEvidence(transactionId) {
  const doc = await db.collection('ady_transactions').doc(transactionId).get();
  const data = doc.data();
  return data.evidenceFiles || [];
}

function displayEvidenceFile(file) {
  // Decode base64
  const byteChars = atob(file.base64Data);
  const byteNumbers = new Array(byteChars.length);
  for (let i = 0; i < byteChars.length; i++) {
    byteNumbers[i] = byteChars.charCodeAt(i);
  }
  const byteArray = new Uint8Array(byteNumbers);
  const blob = new Blob([byteArray], { type: file.fileType });
  const url = URL.createObjectURL(blob);
  
  // Show in modal
  if (file.fileType === 'application/pdf') {
    // Show in iframe
  } else if (file.fileType.startsWith('image/')) {
    // Show as image
  }
  
  // Download button uses the same URL
}
```

---

## 4. Download as PDF

The page should have a **"Download PDF"** button in the sticky header that:

1. Uses `window.print()` with a print-specific CSS stylesheet, OR
2. Uses a library like `html2pdf.js` or `jsPDF` to generate a PDF from the page content

### Print CSS approach (simplest):

```html
<button onclick="window.print()">Download PDF</button>
```

```css
@media print {
  .no-print { display: none; }  /* hide share button, modal, etc. */
  .receipt-button { display: none; }  /* hide clickable receipt buttons in PDF */
  body { font-size: 12pt; }
}
```

### jsPDF approach (better control):

```javascript
import html2pdf from 'html2pdf.js';

function downloadPDF() {
  const element = document.getElementById('report-content');
  const opt = {
    margin: 10,
    filename: 'VentureLab-Report.pdf',
    image: { type: 'jpeg', quality: 0.98 },
    html2canvas: { scale: 2 },
    jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' }
  };
  html2pdf().set(opt).from(element).save();
}
```

---

## 5. Transaction Display Rules

| type | isExpense | isIncome | Display |
|------|-----------|----------|---------|
| cost | true | false | Black text, `-` prefix on amount |
| bill | true | false | Black text, `-` prefix on amount |
| payment | false | true | Green text, `+` prefix on amount |
| check | false | true | Green text, `+` prefix on amount |
| revenue | false | true | Green text, `+` prefix on amount |

Amounts shown in JOD with `JD` prefix, 2 decimal places.

---

## 6. Color Coding

- **Income amounts**: green (#22C55E)
- **Expense amounts**: red (#EF4444) or black
- **Negative cash/balance**: red (#EF4444)
- **Positive cash/balance**: green (#22C55E) or accent blue
- **Debt in carry-over**: red background or red text
- **Surplus in carry-over**: green text

---

## 7. Achievement Icons

| entryType | Icon (use emoji or SVG) |
|-----------|------------------------|
| milestone | 🚩 |
| income | 💰 |
| expense | 💸 |
| allocation | 🔄 |
| research | 🔍 |
| feature | ⭐ |
| bug-fix | 🔧 |
| meeting | 👥 |
| other | • |

---

## 8. Security Notes

- The shared page is **public** — anyone with the token URL can view it
- Evidence files (receipts/bills) are also accessible via the page
- The iOS user can **revoke** the link at any time (sets `revoked: true`)
- The web should check `revoked` on every page load
- Consider adding a note: "This link contains financial data. Do not share publicly."

---

## 9. Implementation Checklist for Web Developer

- [ ] Create route `/shared/:token`
- [ ] Read `sharedViews/{token}` from Firestore on page load
- [ ] Show "link expired" page if `revoked == true` or document missing
- [ ] Render the 7-section Venture Lab report template
- [ ] Show PSUT grant card with progress bar (negative cash in red)
- [ ] Show period summary table with carry-over
- [ ] Show transaction log with ALL transactions (income + expense)
- [ ] Add clickable 📎 button for transactions with `hasEvidence == true`
- [ ] Implement evidence modal that fetches from `ady_transactions` and decodes base64
- [ ] Add "Download PDF" button (using `window.print()` or `html2pdf.js`)
- [ ] Add "Share Link" button (copy URL to clipboard)
- [ ] Make it responsive (mobile + desktop)
- [ ] Style with the same color scheme as the iOS app
