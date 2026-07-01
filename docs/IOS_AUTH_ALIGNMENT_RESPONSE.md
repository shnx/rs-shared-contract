# iOS → Web Team: Auth Alignment Complete

**Date:** July 2, 2026
**From:** iOS team
**Re:** Response to WEB_IOS_ALIGNMENT_NOTES.md

---

## Status: ✅ Fully Aligned

We've read your `WEB_IOS_ALIGNMENT_NOTES.md` and the iOS app now matches your Firebase Auth migration.

### What we implemented

1. **SystemUser model** — added `hasPassword`, `projectIds`, and `permissions` fields to match your `systemUsers` schema. `UserPermissions` struct created with all 10 permission flags.

2. **Password reset** — `AuthManager.sendPasswordReset(email:)` calls `Auth.auth().sendPasswordReset(withEmail:)`. LoginView now has a "Forgot Password?" button.

3. **Username → email lookup** — `AuthManager.emailForUsername(username:)` queries `systemUsers` by username and returns the email. Matches your login flow.

4. **Legacy user migration** — `AuthManager.migrateLegacyUserIfNeeded(email:plaintextPassword:)`:
   - Finds user in `systemUsers` by email
   - If they have a plaintext `password` field but `hasPassword != true`, creates a Firebase Auth account
   - Removes plaintext `password`, sets `hasPassword = true`
   - Called automatically when Firebase sign-in fails with `userNotFound` or `invalidCredentials`

5. **Login flow** — `signIn()` now tries Firebase Auth first, falls back to legacy migration, then retries. This matches your "auto-create on next login" approach.

### What we already had aligned

- Firebase Auth SDK for all authentication ✅
- `systemUsers` Firestore collection for user profiles ✅
- Apple Sign In with Firebase credential ✅
- Admin allowlist by email/username ✅
- No Express backend dependency for auth ✅

### Notes

- iOS login uses **email** directly (not username). Both flows end at `signIn(withEmail:password:)` so this is compatible.
- The `passwordHash` field is kept in the model for backward compatibility but is not used — Firebase Auth handles all password hashing.
- Registration is done through the admin panel (`SettingsView`) and `AdminSetupView`, not a public sign-up page. Both use `Auth.auth().createUser(withEmail:password:)` and write to `systemUsers` with the UID as doc ID.

---

## Venture Reports

We also added a new `venture_reports` Firestore collection for PSUT Venture Lab project reports. See `VENTURE_REPORT_WEB_HANDOFF.md` for details. The web team needs to add a `/r/:token` route to render these reports publicly.
