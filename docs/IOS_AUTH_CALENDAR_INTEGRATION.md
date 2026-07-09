# iOS → Web Team: Auth + Google Calendar Integration

**Date:** July 9, 2026
**From:** iOS team
**Re:** Three login methods working + Google Calendar sync

---

## Auth: Three Login Methods — All Working

The iOS app now supports three login methods, all backed by Firebase Auth:

### 1. Username / Email + Password
- User enters **username or email** + password
- If username (no `@`), app looks up `systemUsers` collection by `username` field → gets email → signs in with `Auth.auth().signIn(withEmail:password:)`
- If email, signs in directly
- Legacy migration: if Firebase Auth fails and user has plaintext `password` field, auto-creates Firebase Auth account
- Password reset works for both username and email entry (resolves username → email first)

### 2. Sign in with Apple
- Uses `ASAuthorizationAppleIDCredential` → Firebase `OAuthProvider.appleCredential`
- Creates/updates `systemUsers` doc with Firebase UID as doc ID
- First user becomes admin, subsequent users become `investor`

### 3. Sign in with Google
- Uses `GoogleSignIn` SDK → Firebase `GoogleAuthProvider.credential`
- Requests `calendar` scope at sign-in time (`additionalScopes: [calendarScope]`)
- Creates/updates `systemUsers` doc with Firebase UID as doc ID

### All three resolve to the same `systemUsers` collection
- Doc ID = Firebase UID (for Apple/Google) or existing doc ID (for legacy username users)
- Email is the primary key for deduplication
- Admin allowlist by email/username is applied on every login

---

## Google Calendar Integration

### How it works

The iOS app can sync bookings with Google Calendar. This works for **all users**, regardless of login method:

#### For Google Sign-In users
- Calendar scope (`https://www.googleapis.com/auth/calendar`) is requested at login time
- Access token is available immediately — no extra steps needed

#### For username/password or Apple Sign In users
- BookingsView shows a **"Connect Google Calendar"** card
- User taps "Connect" → Google Sign-In sheet appears (requests calendar scope only)
- This does **NOT** replace the Firebase auth session — it only gets a Google access token for Calendar API calls
- The user stays logged in with their original method (username/password or Apple)

### Features

1. **Add to Google Calendar** — Creates events on the user's primary Google Calendar for all upcoming bookings. Each event includes:
   - Title: booking service name
   - Description: booking with [name] (email)
   - Start/end time
   - Attendee email (if available)

2. **Import from Calendar** — Fetches events from the user's Google Calendar for the next 30 days. User can:
   - Browse upcoming events (title, date, duration, attendee)
   - Select events with checkboxes
   - Import selected events as bookings into Firestore (`bookings` collection, status: `confirmed`)

### API used
- Google Calendar REST API v3 (`https://www.googleapis.com/calendar/v3/calendars/primary/events`)
- No backend needed — all calls are made directly from the iOS app using the Google access token
- Token refresh is handled automatically by the GoogleSignIn SDK

---

## What the Web Team Should Do

### 1. Web login should also support all three methods
- **Username/email + password**: Already aligned (see `IOS_AUTH_ALIGNMENT_RESPONSE.md`)
- **Google Sign-In**: Web should add Google Sign-In via Firebase Auth JS SDK
  - Request `calendar` scope for calendar features
- **Apple Sign-In**: Web should add Sign in with Apple via Firebase Auth JS SDK

### 2. Web can add Google Calendar features (optional)
If the web team wants to add calendar sync on the web:
- Use Google Identity Services for web to get an access token with calendar scope
- Call the same Google Calendar REST API
- Or use a Cloud Function to create calendar events (server-side with service account)

### 3. Firestore `bookings` collection
- Imported calendar events are saved with `serviceType: "imported"`, `status: "confirmed"`
- Web admin panel should display these alongside website bookings
- No schema changes needed — uses existing `Booking` model

### 4. Firebase Console
- Ensure **Google** and **Apple** sign-in providers are enabled in Firebase Auth
- No additional configuration needed for calendar — the scope is handled client-side

---

## Files Changed (iOS)

| File | Change |
|------|--------|
| `LoginView.swift` | Username/email field, all three login methods, password reset for username |
| `AuthManager.swift` | Already had Google + Apple user creation (no changes needed) |
| `GoogleCalendarService.swift` | New — Calendar REST API, `connectGoogleCalendar()` for non-Google users |
| `CalendarImportView.swift` | New — Browse/select/import Google Calendar events as bookings |
| `BookingsView.swift` | Calendar connect card, "Add to Calendar" + "Import from Calendar" buttons |

---

## Setup Requirements (already documented)

See `GOOGLE_SIGNIN_SETUP.md` for Xcode package + URL scheme setup. The GoogleSignIn SDK must be added to the Xcode project for calendar features to work. If the SDK is not present, calendar features gracefully degrade (show error message), and username/password + Apple sign-in still work.
