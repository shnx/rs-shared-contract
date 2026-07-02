# Google Sign-In Setup for iOS App

The Swift code is ready — `LoginView.swift` and `AuthManager.swift` both have Google Sign-In support behind `#if canImport(GoogleSignIn)`. You just need to add the SDK and URL scheme in Xcode.

## Step 1: Add GoogleSignIn Package

1. Open `rs.xcworkspace` in Xcode
2. Select the **rs** project in the navigator
3. Go to **File → Add Package Dependencies…**
4. Enter: `https://github.com/google/GoogleSignIn-iOS`
5. Select **GoogleSignIn** (and **GoogleSignInSwift** if you want the SwiftUI button)
6. Add to the **rs** target

## Step 2: Add URL Scheme

Google Sign-In requires a reversed client ID URL scheme so the Google app can redirect back.

1. Select the **rs** project → **rs** target → **Info** tab
2. Expand **URL Types** → click **+**
3. **URL Scheme**: `com.googleusercontent.apps.531306594290-a6f79g782krg2cnkqnmf6oee3ju3pncp`
   (This is the `REVERSED_CLIENT_ID` from your `GoogleService-Info.plist`)

## Step 3: Firebase Console

1. Go to [Firebase Console](https://console.firebase.google.com) → `robotics-website-5593f`
2. **Authentication → Sign-in method**
3. Enable **Google** if not already enabled
4. Under **Authorized domains**, make sure your app's bundle ID `com.the-rs.rs` is supported
5. Download the updated `GoogleService-Info.plist` if any changes were made

## That's it!

Build and run. The login screen will now show a "Sign in with Google" button below the Apple Sign-In button.
