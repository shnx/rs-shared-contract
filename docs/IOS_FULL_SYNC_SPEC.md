# iOS → Web Team: Full Sync Specification

**Date:** July 9, 2026
**From:** iOS team
**Re:** Everything the web portal needs to sync 100% with the iOS app

---

## Firebase Project

- **Project ID:** `robotics-website-5593f`
- Both apps share the same Firestore + Firebase Auth

---

## 1. Authentication

### Three login methods (all working on iOS)

| Method | How it works |
|--------|-------------|
| Username/Email + Password | User enters username OR email. If username, iOS looks up `systemUsers` by `username` → gets email → `Auth.auth().signIn(withEmail:password:)` |
| Sign in with Apple | `ASAuthorizationAppleIDCredential` → Firebase `OAuthProvider.appleCredential` |
| Sign in with Google | `GoogleSignIn` SDK → Firebase `GoogleAuthProvider.credential`. Requests calendar scope at login |

### `systemUsers` collection

```jsonc
{
  "id": "string",           // = Firebase UID (doc ID)
  "username": "string",     // unique, used for username login
  "passwordHash": "string?",// deprecated — Firebase Auth handles passwords
  "password": "string?",    // legacy plaintext (migrated on first login)
  "hasPassword": "boolean?",// true after Firebase Auth account created
  "role": "admin | manager | investor",
  "displayName": "string?",
  "name": "string?",
  "email": "string",        // primary key for dedup
  "phone": "string?",
  "projectIds": ["string"]?,
  "permissions": {
    "canViewDashboard": "boolean?",
    "canViewFinancials": "boolean?",
    "canViewProjectManagement": "boolean?",
    "canViewCRM": "boolean?",
    "canViewCommunity": "boolean?",
    "canViewBuilders": "boolean?",
    "canEditOwnData": "boolean?",
    "canManageUsers": "boolean?",
    "canManageContent": "boolean?",
    "canManageProjects": "boolean?"
  },
  "createdAt": "Timestamp?",
  "updatedAt": "Timestamp?"
}
```

### Auth flow notes for web
- **Legacy migration:** If Firebase Auth fails and user has plaintext `password` field, iOS auto-creates Firebase Auth account, sets `hasPassword=true`, removes plaintext. Web should do the same.
- **Admin allowlist:** Emails `mohammedshannak@gmail.com` and username `shannak_m` are always admin.
- **First user rule:** If `systemUsers` is empty, first sign-in (any method) becomes admin. Subsequent users get `investor` role.
- **Password reset:** `Auth.auth().sendPasswordReset(withEmail:)`. iOS resolves username→email first.

### Google Calendar (iOS only, web optional)
- iOS requests scope `https://www.googleapis.com/auth/calendar` at Google login
- For username/Apple users, iOS shows "Connect Google Calendar" card → Google Sign-In for calendar scope only (doesn't change Firebase session)
- iOS calls Google Calendar REST API v3 directly: `GET/POST https://www.googleapis.com/calendar/v3/calendars/primary/events`
- **Web can add this too** using Google Identity Services JS library

---

## 2. Collection Prefix Logic

iOS uses `ady_` prefix for its own writes. Web writes to **un-prefixed** collections. iOS reads both and merges.

| Un-prefixed (web) | Prefixed (iOS) | Merge Strategy |
|---|---|---|
| `transactions` | `ady_transactions` | Dedup by ID, keep newer `updatedAt` |
| `periods` | `ady_periods` | Dedup by ID, keep newer `updatedAt` |
| `documents` | `ady_documents` | Try un-prefixed first, fall back to prefixed |

All other collections below are **un-prefixed** (shared directly).

---

## 3. Shared Collections — Full Schema

### `consultations` (service catalog)

```jsonc
{
  "id": "string",
  "title": "string",
  "description": "string",
  "tier": "first_coffee | mentoring | professional | company | workshop | advisor",
  "category": "training | advisory | workshop | technical | career",
  "durationMinutes": "number?",
  "price": "number?",         // base price
  "currency": "JOD | USD | EUR?",
  "targetAudience": "student | professional | company",
  "deliverables": ["string"]?,
  "isActive": "boolean?",
  "createdBy": "string?",
  "createdAt": "Timestamp?",
  "updatedAt": "Timestamp?"
}
```

### `bookings` (shared — web writes, iOS reads + writes)

```jsonc
{
  "id": "string",
  "userId": "string?",
  "email": "string",          // required
  "name": "string",           // required
  "company": "string?",
  "consultationId": "string?",// ref to consultations
  "serviceOfferingId": "string?", // legacy
  "serviceType": "string?",   // "first_coffee", "mentoring", "imported", etc.
  "tier": "string?",          // denormalized from consultation
  "scheduledAt": "Timestamp?",
  "durationMinutes": "number?",
  "status": "pending | confirmed | completed | cancelled",
  "priceUSD": "number?",
  "priceEUR": "number?",
  "stripeSessionId": "string?",
  "paymentStatus": "unpaid | paid",
  "paymentMethod": "stripe | manual | invoice",
  "invoiceUrl": "string?",
  "notes": "string?",
  "createdAt": "Timestamp?"
}
```

**iOS writes to this collection** when importing Google Calendar events. Imported events have `serviceType: "imported"`, `status: "confirmed"`.

### `milestones` (iOS writes, web can read)

```jsonc
{
  "id": "string",
  "title": "string",
  "description": "string?",
  "category": "ecosystem | product | marketing | technical | business | community | research",
  "status": "planned | in_progress | completed | achieved",
  "date": "Timestamp?",
  "tags": ["string"]?,
  "notes": "string?",
  "aiGenerated": "boolean?",
  "relatedIds": ["string"]?,
  "createdBy": "string?",
  "createdAt": "Timestamp?",
  "updatedAt": "Timestamp?"
}
```

### `crm` (iOS writes, web can read)

```jsonc
{
  "id": "string",
  "name": "string",
  "email": "string?",
  "phone": "string?",
  "source": "string?",
  "status": "new | interested | qualified | demo-promised | active | won | renewal-needed | rejected | lost",
  "notes": "string?",
  "ownerUserId": "string?",
  "company": "string?",
  "value": "number?",
  "isBuilder": "boolean?",
  "leadScores": {
    "technical": "number?",       // 0-100
    "curiosity": "number?",
    "communication": "number?",
    "consistency": "number?",
    "initiative": "number?",
    "portfolio": "number?",
    "availability": "number?",
    "leadership": "number?",
    "entrepreneurship": "number?"
  },
  "badges": ["string"]?,
  "createdAt": "Timestamp?",
  "updatedAt": "Timestamp?"
}
```

### `crm_contacts` (iOS writes via SubmissionResponseView)

```jsonc
{
  "id": "string",
  "submissionId": "string?",
  "name": "string?",
  "email": "string?",
  "phone": "string?",
  "role": "string?",
  "status": "string?",
  "proposedCourses": ["string"]?,
  "enrolledCourses": ["string"]?,
  "proposedServices": ["string"]?,
  "cvFileName": "string?",
  "cvNotes": "string?",
  "accountCreated": "boolean?",
  "accountId": "string?",
  "emailSentAt": "Timestamp?",
  "responseMessage": "string?",
  "createdAt": "Timestamp?",
  "responseToken": "string?",
  "lastResponseType": "string?",
  "lastResponseAt": "Timestamp?",
  "isBuilder": "boolean?",
  "leadScores": "LeadScores?",  // same structure as crm
  "badges": ["string"]?,
  "assessmentAnswers": ["string"]?
}
```

### `builder_portfolios`

```jsonc
{
  "id": "string",
  "builderId": "string",
  "builderName": "string?",
  "builderEmail": "string?",
  "title": "string",
  "description": "string?",
  "projectUrl": "string?",
  "images": ["string"]?,
  "tags": ["string"]?,       // e.g., ["ROS", "Computer Vision"]
  "skills": ["string"]?,
  "featured": "boolean?",
  "views": "number?",
  "likes": "number?",
  "createdAt": "Timestamp?",
  "updatedAt": "Timestamp?"
}
```

### `talent_matches`

```jsonc
{
  "id": "string",
  "companyId": "string?",
  "companyName": "string?",
  "requirements": "string?",
  "skills": ["string"]?,
  "matchedBuilderIds": ["string"]?,
  "status": "open | in_progress | filled | closed",
  "createdAt": "Timestamp?",
  "updatedAt": "Timestamp?"
}
```

### `monthly_content`

```jsonc
{
  "id": "string",
  "month": "string?",           // "2026-07"
  "papers": [{"title": "string?", "url": "string?", "description": "string?", "tags": ["string"]?}]?,
  "tools": ["ContentItem"]?,
  "internships": ["ContentItem"]?,
  "challenges": ["ContentItem"]?,
  "cvTips": ["ContentItem"]?,
  "manufacturingInsights": ["ContentItem"]?,
  "startupLessons": ["ContentItem"]?,
  "sentAt": "Timestamp?",
  "createdAt": "Timestamp?"
}
```

### `project_challenges`

```jsonc
{
  "id": "string",
  "title": "string",
  "description": "string?",
  "theme": "string?",
  "requirements": ["string"]?,
  "prizes": ["string"]?,
  "sponsor": "string?",
  "startDate": "Timestamp?",
  "endDate": "Timestamp?",
  "status": "upcoming | active | closed",
  "submissions": ["string"]?,  // builder portfolio IDs
  "winners": ["string"]?,      // builder IDs
  "createdAt": "Timestamp?",
  "updatedAt": "Timestamp?"
}
```

### `job_submissions` (web writes, iOS reads)

iOS tries these collection names in order: `job_submissions`, `jobSubmissions`, `submissions`, `applications`, `careerApplications`, `career_applications`, `cvSubmissions`, `cv_submissions`.

```jsonc
{
  "id": "string",
  "applicantName": "string?",  "name": "string?",  "fullName": "string?",
  "email": "string?",  "phone": "string?",
  "resumeUrl": "string?",  "cvUrl": "string?",
  "position": "string?",  "role": "string?",
  "message": "string?",  "coverLetter": "string?",
  "status": "new | reviewed | responded | invited | interested | declined | survey_completed | enrolled | rejected",
  "submittedAt": "Timestamp?",  "createdAt": "Timestamp?",
  "isGraduate": "boolean?",  "jobType": "string?",
  "cvFileName": "string?",  "cvFileType": "string?",  "cvFileSize": "number?",
  "cvBase64": "string?",  "extractedText": "string?",
  "careerId": "string?",  "acceptsUnpaidInternship": "boolean?",
  "notes": "string?",  "subject": "string?",
  "reviewedAt": "Timestamp?",  "reviewedBy": "string?",
  "invitedAt": "Timestamp?",  "invitedBy": "string?",
  "deviceInfo": "DeviceInfo?",
  "aiAnalysis": "CVAnalysis?",
  "extractedData": "CVExtractedData?",
  "preparedCVIds": ["string"]?,
  "offerSentAt": "Timestamp?",  "offerType": "string?",  "offerToken": "string?"
}
```

### `contact_submissions` (web writes, iOS reads)

iOS tries: `contact_submissions`, `contact_messages`, `contactMessages`, `inquiries`, `contactForms`, `contact_forms`, `messages`.

```jsonc
{
  "id": "string",
  "name": "string?",  "fullName": "string?",
  "email": "string?",  "phone": "string?",
  "subject": "string?",  "message": "string?",  "body": "string?",
  "company": "string?",  "notes": "string?",
  "submittedAt": "Timestamp?",  "createdAt": "Timestamp?",
  "status": "string?",
  "deviceInfo": "DeviceInfo?"
}
```

### `career_opportunities` (web writes, iOS reads)

iOS tries: `career_opportunities`, `careers`.

```jsonc
{
  "id": "string",
  "title": "string?",  "titleAr": "string?",
  "department": "string?",  "location": "string?",  "type": "string?",
  "description": "string?",  "descriptionAr": "string?",
  "isActive": "boolean?",  "aiFormatted": "boolean?",
  "createdAt": "Timestamp?",  "updatedAt": "Timestamp?"
}
```

### `courses` (web writes, iOS reads)

```jsonc
{
  "id": "string",
  "title": "string?",  "titleAr": "string?",
  "description": "string?",  "category": "string?",
  "durationWeeks": "number?",  "level": "string?",
  "price": "number?",  "currency": "string?",
  "isActive": "boolean?",  "isPublished": "boolean?",  "isFree": "boolean?",
  "instructor": "string?",  "lectureCount": "number?",
  "createdAt": "Timestamp?"
}
```

### `email_responses` (iOS reads)

```jsonc
{
  "id": "string",
  "contactId": "string?",  "email": "string?",
  "responseType": "accept | decline | survey",
  "submittedAt": "Timestamp?",  "createdAt": "Timestamp?",
  "answers": ["string"]?
}
```

### `email_inbox` (email-to-expense system)

```jsonc
{
  "id": "string",
  "messageId": "string?",  "from": "string?",  "fromName": "string?",
  "to": "string?",  "subject": "string?",
  "bodyText": "string?",  "bodyHtml": "string?",
  "receivedAt": "Timestamp?",  "processedAt": "Timestamp?",
  "createdAt": "Timestamp?",
  "status": "pending_review | approved | rejected | ignored",
  "attachments": [{"filename": "string?", "mimeType": "string?", "size": "number?", "pdfText": "string?"}]?,
  "extractedData": {
    "amount": "number?",  "currency": "string?",  "date": "string?",
    "vendor": "string?",  "description": "string?",  "category": "string?",
    "confidence": "number?"
  },
  "transactionId": "string?",
  "reviewedBy": "string?",  "reviewedAt": "Timestamp?"
}
```

### `periodSummaries` (iOS writes)

```jsonc
{
  "id": "string",
  "projectId": "string?",  "periodId": "string?",  "monthKey": "string?",
  "label": "string",
  "totalExpense": "number",  "totalIncome": "number",  "netPosition": "number",
  "transactionCount": "number",
  "byType": [{"type": "string", "total": "number", "count": "number"}],
  "byCategory": [{"category": "string", "planned": "number", "actual": "number"}],
  "computedAt": "Timestamp"
}
```

### `sharedViews` (iOS writes)

```jsonc
{
  "id": "string",
  "projectId": "string",  "createdBy": "string",
  "token": "string",      // UUID for URL
  "label": "string?",  "expiresAt": "Timestamp?",
  "createdAt": "Timestamp",  "revoked": "boolean"
}
```

### `ady_workSessions` (iOS writes)

```jsonc
{
  "id": "string",
  "userId": "string",  "userName": "string",
  "startTime": "Timestamp",  "endTime": "Timestamp?",
  "isActive": "boolean",  "notes": "string?",
  "projectId": "string?",  "projectName": "string?",
  "breakDuration": "number",  // seconds
  "location": "string?",
  "createdAt": "Timestamp",  "updatedAt": "Timestamp"
}
```

### `ady_employeePermissions` (iOS writes)

```jsonc
{
  "id": "string",
  "userId": "string",  "grantedBy": "string",
  "projectIds": ["string"],
  "canViewStores": "boolean",  "canViewFinancials": "boolean",
  "canViewCRM": "boolean",  "canEditOwnData": "boolean",
  "createdAt": "Timestamp",  "updatedAt": "Timestamp"
}
```

### `ady_meetings` (meeting notes with Sari)

```jsonc
{
  "id": "string",
  "projectId": "string?",  "periodId": "string?",
  "meetingDate": "Timestamp?",
  "mainPoints": "string?",  "decisions": "string?",
  "createdBy": "string?",  "createdAt": "Timestamp?"
}
```

### `ady_pdf_meta` (OCR data, iOS writes)

```jsonc
{
  "id": "string",
  "projectId": "string?",  "transactionId": "string?",
  "fileName": "string?",
  "amount": "number?",  "currency": "string?",  "date": "Timestamp?",
  "vendor": "string?",  "invoiceNumber": "string?",
  "suggestedCategory": "string?",  "suggestedProject": "string?",
  "confidence": "number?",  "method": "on-device",
  "extractedText": "string?",  "rawJSON": "string?",
  "analyzedAt": "Timestamp?"
}
```

---

## 4. Computed Values (must match)

### Currency
- JOD = as-is
- USD → JOD = multiply by `0.707`

### Transaction classification
- **Expense** = `type == "cost"` OR `type == "bill"`
- **Income** = `type == "revenue"` OR `type == "payment"` OR `type == "check"`

### Period status decoding
| Stored | Decoded as |
|---|---|
| `planned` | planned |
| `completed`, `closed`, `done` | completed |
| anything else | active |

### Fund total
Hardcoded: 10,000 JOD (stored in `@AppStorage` on iOS)

---

## 5. Cloud Functions (specs ready)

See `CLOUD_FUNCTIONS_SPEC.md` for full details:

1. **`sendWelcomeToCommunity`** — `onCreate` on `crm_contacts` → welcome email
2. **`sendMonthlyCommunityDigest`** — Scheduled monthly → digest from `monthly_content`
3. **`calculateLeadScores`** — `onUpdate` on `crm_contacts` → auto-calculate 9-dimension scores + badges

---

## 6. What Web Team Needs to Build

### Auth
- [ ] Support username login (look up `systemUsers.username` → get email → Firebase Auth)
- [ ] Add Sign in with Apple (Firebase Auth JS SDK)
- [ ] Add Sign in with Google (Firebase Auth JS SDK)
- [ ] Legacy migration: auto-create Firebase Auth account for users with plaintext `password`
- [ ] Password reset working for both username and email entry

### Collections
- [ ] Write to `consultations` with the schema above (6 tiers)
- [ ] Write to `bookings` with `consultationId`, `tier`, `paymentMethod`, `invoiceUrl`
- [ ] Read `milestones` (iOS creates these via AI chat)
- [ ] Read `crm` and `crm_contacts` for lead scores and builder badges
- [ ] Read `builder_portfolios` for builder showcase
- [ ] Write to `monthly_content` for community digest
- [ ] Write to `project_challenges` for hackathon challenges
- [ ] Read `ady_meetings` for meeting notes with Sari

### Booking flow
- [ ] Stripe Checkout for student payments (EUR)
- [ ] Manual invoice option for companies (USD)
- [ ] Booking page at `/book` with consultation catalog
- [ ] Assessment email after First Coffee (questions: excitement, blockers, 1-year vision)
- [ ] `offerSentAt`, `offerType`, `offerToken` fields on `job_submissions`

### Optional: Google Calendar on web
- [ ] Use Google Identity Services JS for calendar scope
- [ ] Call same Google Calendar REST API
- [ ] Or use Cloud Function with service account for server-side calendar events

---

## 7. Pricing Tiers (confirmed)

| Tier | Price | Duration | Audience |
|------|-------|----------|----------|
| First Coffee | €15 | 30 min | Student |
| Mentoring | €60-120/hr | 60 min | Student/Pro |
| Professional | €200-340/hr | 90 min | Professional |
| Company | €350-600/session | 2-4 hrs | Company |
| Workshop | €500-1000 | Half/full day | Group |
| Advisor | €1000-5000/month | Retainer | Company |

**Currency rule:** EUR for students, USD for companies. Store both `priceEUR` and `priceUSD`.

---

This document supersedes all previous partial specs. Any questions, ping us.
