# Web ↔ iOS Alignment Notes

**Date:** July 1, 2026  
**From:** Web team (Mohammad Shannak)  
**To:** iOS team  
**Re:** Firebase Auth migration and cross-platform alignment

---

## Summary

We've completed the **Firebase Auth migration** on the web side. This directly aligns with your `devin/1782426331-firebase-alignment` and `devin/1782472467-firebase-alignment` branches. The Express backend is no longer required for authentication.

---

## What We Changed

### 1. Authentication → Firebase Auth SDK

**Before:** Custom Express backend with bcrypt hashing at `localhost:3001/api/auth/login`  
**After:** Firebase Auth SDK (`signInWithEmailAndPassword`, `createUserWithEmailAndPassword`)

- **Login flow:** Look up `username` in Firestore `systemUsers` collection → get `email` → call `signInWithEmailAndPassword`
- **Registration:** `createUserWithEmailAndPassword` → create Firestore profile with UID as doc ID
- **Password reset:** `sendPasswordResetEmail` (Firebase sends the email natively)
- **Session persistence:** `onAuthStateChanged` listener + localStorage cache
- **Legacy migration:** If a user has a plaintext password in Firestore but no Firebase Auth account, we auto-create one on their next login and remove the plaintext password

### 2. User Management → Firebase Auth + Firestore

- **Create user:** `createUserWithEmailAndPassword(auth, email, password)` → `setDoc(doc(db, 'systemUsers', uid), profile)`
- **Edit user:** `updateDoc` on Firestore `systemUsers` document
- **Delete user:** Deletes Firestore profile (Firebase Auth account deletion requires Admin SDK — noted in UI)
- **Password changes:** Firebase Auth requires the user to be signed in or Admin SDK. Admin edits show a note: "User must use Forgot Password or set it on next login"

### 3. Impersonation (Admin "Log in as user")

- **Pure client-side:** Admin fetches target user's Firestore profile → swaps `currentUser` in React context → stores admin user in `localStorage('portal_impersonating')`
- **Stop impersonation:** Restores admin from localStorage
- **No backend needed** — works entirely through Firestore reads

### 4. Forgot Password / Forgot Username

- **Forgot password:** `firebaseAuth.sendPasswordReset(email)` — Firebase sends reset link
- **Forgot username:** Queries `systemUsers` collection by email, returns `username` field

### 5. "Must Set Credentials" Flow

- New `SetCredentialsPage.tsx` — users without `hasPassword: true` are redirected here
- Uses `firebaseAuth.updatePassword()` (requires recent login)
- Updates `username` in Firestore profile

### 6. Invitation Link Flow

- `/register?invite=X` → routes to `StudentOnboardingPage` (onboarding + account setup)
- `/register` (no invite) → routes to `CandidateRegisterPage` (standard CV submission)

---

## Shared Schema (systemUsers collection)

```json
{
  "id": "Firebase Auth UID",
  "username": "string (lowercase)",
  "email": "string (lowercase)",
  "name": "string",
  "role": "admin | student | candidate | advisor | client",
  "hasPassword": "boolean",
  "projectIds": "string[]",
  "permissions": {
    "canViewDashboard": "boolean",
    "canViewFinancials": "boolean",
    "canViewProjectManagement": "boolean",
    "canViewCRM": "boolean",
    "canViewContracts": "boolean",
    "canViewTransactions": "boolean",
    "canViewReports": "boolean",
    "canEdit": "boolean",
    "canDelete": "boolean",
    "canManageUsers": "boolean"
  },
  "createdAt": "ISO 8601"
}
```

**Note:** We no longer store `password` or `passwordHash` in Firestore. Password hashing is handled entirely by Firebase Auth.

---

## Firebase Project

- **Project ID:** `robotics-website-5593f` (aligned with your branches)
- **Auth Domain:** `robotics-website-5593f.firebaseapp.com`
- **Config:** Shared via `.env` with `VITE_FIREBASE_*` env vars (same values work for iOS `GoogleService-Info.plist`)

---

## What Still Uses the Express Server

The Express server (`server/server.js`) is still used for:
- Google Sheets export/import (`/api/v1/sheets/*`)
- LaTeX compilation (`/api/latex/*`)
- File-based JSON database (legacy, being phased out)

**Authentication no longer touches the Express server.** You can safely use Firebase iOS SDK for all auth flows.

---

## Action Items for iOS Team

1. **Use Firebase iOS SDK** (`FirebaseAuth`, `FirebaseFirestore`) — same project, same collections
2. **Login:** Query `systemUsers` by `username` → get `email` → `Auth.auth().signIn(withEmail:email, password:password)`
3. **Register:** `Auth.auth().createUser(withEmail:email, password:password)` → write profile to `systemUsers` with UID as doc ID
4. **Password reset:** `Auth.auth().sendPasswordReset(withEmail:email)` — Firebase handles the email
5. **Legacy users:** If `systemUsers` doc has a `password` field (plaintext), migrate by creating Firebase Auth account on first login
6. **CRM lead statuses:** We've noted your expanded 9-value enum (`new, interested, qualified, demo-promised, active, won, renewal-needed, rejected, lost`) — we'll merge that from your branch

---

## Questions?

Contact: Mohammad Shannak (mohammedshannak@gmail.com)
