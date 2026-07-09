# Cloud Functions Specifications

## Overview

Three Cloud Functions to support the builder ecosystem: welcome email, monthly digest, and automated lead scoring.

---

## 1. `sendWelcomeToCommunity`

**Trigger:** `onCreate` on `crm_contacts` collection

**Purpose:** When a new contact is created (via CV submission, consultation booking, or manual CRM entry), send a personalized welcome email and set `isBuilder` flag if applicable.

### Logic

```typescript
exports.sendWelcomeToCommunity = functions.firestore
  .document('crm_contacts/{contactId}')
  .onCreate(async (snap, context) => {
    const contact = snap.data();
    
    // Skip if already welcomed
    if (contact.welcomeSentAt) return;
    
    // Determine email template based on source
    const isBuilder = contact.isBuilder === true;
    const hasAssessment = contact.assessmentAnswers?.length > 0;
    
    const template = isBuilder
      ? 'welcome-builder'
      : hasAssessment
        ? 'welcome-assessed'
        : 'welcome-general';
    
    // Send email via SendGrid / Mailgun / Firebase Extensions
    await sendEmail({
      to: contact.email,
      template,
      data: {
        name: contact.name,
        isBuilder,
        // Include link to book First Coffee
        bookingUrl: `https://the-rs.com/consultations/first-coffee`,
        // Include community link
        communityUrl: 'https://the-rs.com/community',
      },
    });
    
    // Update contact
    await snap.ref.update({
      welcomeSentAt: admin.firestore.FieldValue.serverTimestamp(),
    });
  });
```

### Email Templates

- **welcome-builder**: "Welcome to the Builder Ecosystem" — mentions portfolio upload, challenges, mentoring
- **welcome-assessed**: "Your Assessment Results" — includes lead score summary, recommends next steps
- **welcome-general**: "Welcome to Robotics Science" — introduces ecosystem, suggests First Coffee

---

## 2. `sendMonthlyCommunityDigest`

**Trigger:** Scheduled Cloud Function (cron), 1st of every month at 09:00 UTC

**Purpose:** Send the monthly content digest to all community members who opted in.

### Logic

```typescript
exports.sendMonthlyCommunityDigest = functions.pubsub
  .schedule('0 9 1 * *')
  .timeZone('UTC')
  .onRun(async (context) => {
    const monthKey = new Date().toISOString().slice(0, 7); // "2026-07"
    
    // Fetch monthly content
    const contentDoc = await admin.firestore()
      .collection('monthly_content')
      .doc(monthKey)
      .get();
    
    if (!contentDoc.exists) {
      console.log(`No content for ${monthKey}, skipping digest`);
      return;
    }
    
    const content = contentDoc.data();
    
    // Get all subscribed contacts
    const contactsSnap = await admin.firestore()
      .collection('crm_contacts')
      .where('digestOptIn', '==', true)
      .get();
    
    const batchSize = 100;
    const batches = chunk(contactsSnap.docs, batchSize);
    
    for (const batch of batches) {
      const emails = batch
        .map(doc => doc.data())
        .filter(c => c.email)
        .map(c => ({
          to: c.email,
          name: c.name,
        }));
      
      await sendBulkEmail({
        template: 'monthly-digest',
        recipients: emails,
        data: {
          month: formatMonth(monthKey),
          papers: content.papers,
          tools: content.tools,
          internships: content.internships,
          challenges: content.challenges,
          cvTips: content.cvTips,
          manufacturingInsights: content.manufacturingInsights,
          startupLessons: content.startupLessons,
        },
      });
    }
    
    // Mark as sent
    await contentDoc.ref.update({
      sentAt: admin.firestore.FieldValue.serverTimestamp(),
    });
    
    console.log(`Digest sent to ${contactsSnap.size} contacts`);
  });
```

### Monthly Content Document Schema

```
monthly_content/{monthKey}
  - month: "2026-07"
  - papers: [{ title, url, description, tags }]
  - tools: [{ title, url, description, tags }]
  - internships: [{ title, url, description, tags }]
  - challenges: [{ title, url, description, tags }]
  - cvTips: [{ title, url, description, tags }]
  - manufacturingInsights: [{ title, url, description, tags }]
  - startupLessons: [{ title, url, description, tags }]
  - sentAt: timestamp
  - createdAt: timestamp
```

---

## 3. `calculateLeadScores`

**Trigger:** `onUpdate` on `crm_contacts` collection (when assessment answers, portfolio, or booking data changes)

**Purpose:** Automatically calculate or update lead scores based on activity, assessment answers, portfolio quality, and engagement.

### Scoring Dimensions (0-100 each)

| Dimension | Source Signals |
|-----------|---------------|
| `technical` | CV skills match, GitHub repos, project complexity |
| `curiosity` | Assessment answer depth, questions asked in First Coffee |
| `communication` | Email response rate, clarity of CV, interview notes |
| `consistency` | Booking attendance, follow-through on action items |
| `initiative` | Self-started projects, challenge participation, proactive outreach |
| `portfolio` | Number of projects, quality score, GitHub stars, demo quality |
| `availability` | Response time, calendar flexibility, stated availability |
| `leadership` | Team lead experience, mentorship activity, community contributions |
| `entrepreneurship` | Startup experience, business ideas, revenue-generating projects |

### Logic

```typescript
exports.calculateLeadScores = functions.firestore
  .document('crm_contacts/{contactId}')
  .onUpdate(async (change, context) => {
    const before = change.before.data();
    const after = change.after.data();
    
    // Only recalculate if relevant fields changed
    const relevantFields = ['assessmentAnswers', 'isBuilder', 'badges', 'portfolioIds'];
    const hasRelevantChange = relevantFields.some(
      field => JSON.stringify(before[field]) !== JSON.stringify(after[field])
    );
    
    if (!hasRelevantChange) return;
    
    // Skip if scores were manually set recently (within 24h)
    if (after.leadScores?.manuallySetAt) {
      const age = Date.now() - after.leadScores.manuallySetAt.toMillis();
      if (age < 24 * 60 * 60 * 1000) return;
    }
    
    const scores = {
      technical: calculateTechnicalScore(after),
      curiosity: calculateCuriosityScore(after),
      communication: calculateCommunicationScore(after),
      consistency: calculateConsistencyScore(after),
      initiative: calculateInitiativeScore(after),
      portfolio: calculatePortfolioScore(after),
      availability: calculateAvailabilityScore(after),
      leadership: calculateLeadershipScore(after),
      entrepreneurship: calculateEntrepreneurshipScore(after),
    };
    
    // Auto-assign badges based on scores
    const badges = generateBadges(scores, after);
    
    // Determine if contact qualifies as builder
    const overallScore = Object.values(scores).reduce((a, b) => a + b, 0) / 9;
    const shouldbeBuilder = overallScore >= 50 || after.isBuilder === true;
    
    await change.after.ref.update({
      leadScores: scores,
      badges,
      isBuilder: shouldBeBuilder,
      scoresUpdatedAt: admin.firestore.FieldValue.serverTimestamp(),
    });
  });
```

### Badge Generation Rules

| Badge | Condition |
|-------|-----------|
| `Coffee Drinker` | Has a completed First Coffee booking |
| `Mentee` | Has attended at least 1 mentoring session |
| `Challenge Winner` | Won a project challenge |
| `Challenge Participant` | Participated in a project challenge |
| `Contributor` | Contributed to open-source or community projects |
| `Connector` | Referred 2+ people to the ecosystem |
| `Rising Star` | Overall score ≥ 75 and improving trend |
| `Builder` | Registered builder with portfolio |

### Scoring Helper Functions

```typescript
function calculateTechnicalScore(contact): number {
  let score = 30; // baseline
  if (contact.assessmentAnswers?.length >= 3) score += 20;
  if (contact.cvNotes?.match(/ROS|Python|C\+\+|TensorFlow|PyTorch/i)) score += 15;
  if (contact.portfolioIds?.length >= 3) score += 20;
  if (contact.badges?.includes('Challenge Winner')) score += 15;
  return Math.min(score, 100);
}

function calculateCuriosityScore(contact): number {
  let score = 30;
  if (contact.assessmentAnswers?.some(a => a.length > 100)) score += 25;
  if (contact.bookings?.some(b => b.serviceType === 'first_coffee')) score += 20;
  if (contact.digestOptIn) score += 15;
  if (contact.badges?.includes('Connector')) score += 10;
  return Math.min(score, 100);
}

// ... similar implementations for other dimensions
```

---

## Deployment

```bash
# Install Firebase CLI
npm install -g firebase-tools

# Initialize functions
firebase init functions

# Deploy
firebase deploy --only functions:sendWelcomeToCommunity
firebase deploy --only functions:sendMonthlyCommunityDigest
firebase deploy --only functions:calculateLeadScores

# Or deploy all
firebase deploy --only functions
```

### Required Environment Variables

```bash
firebase functions:secrets:set SENDGRID_API_KEY
firebase functions:secrets:set MAILGUN_API_KEY
firebase functions:secrets:set COMMUNITY_FROM_EMAIL
```

### Firestore Security Rules Addition

```
match /monthly_content/{monthId} {
  allow read: if true;  // public read for community members
  allow write: if isAdmin();
}

match /milestones/{milestoneId} {
  allow read: if isAuthenticated();
  allow create: if isAuthenticated();
  allow update, delete: if isAdmin();
}
```
