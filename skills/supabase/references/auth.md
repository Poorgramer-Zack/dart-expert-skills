---
name: "supabase-auth"
description: "Expert guide for Supabase Authentication covering auth state management, email/password, OTP/Magic Link, phone, and native social sign-in flows (Google v7, Apple with nonce) using signInWithIdToken."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Supabase Authentication

## Goal
Securely authenticate users with Supabase. The golden rule is routing all navigation decisions through the `onAuthStateChange` stream. Native social sign-ins (Google, Apple) use a **native SDK → ID Token → `signInWithIdToken`** bridge pattern — never redirect to a WebView for these on mobile.

---

## Instructions

### Auth State (The Single Source of Truth)
```dart
final supabase = Supabase.instance.client;

// Listen globally — usually in a Riverpod StreamProvider at app root
supabase.auth.onAuthStateChange.listen((data) {
  final AuthChangeEvent event = data.event;
  final Session? session = data.session;

  switch (event) {
    case AuthChangeEvent.signedIn:
      // Navigate to Home
    case AuthChangeEvent.signedOut:
      // Navigate to Login
    case AuthChangeEvent.tokenRefreshed:
      // Token refreshed silently — no action needed
    case AuthChangeEvent.userUpdated:
      // User profile updated
    default:
      break;
  }
});
```

---

### Email & Password
```dart
// Sign-up
await supabase.auth.signUp(
  email: 'user@email.com',
  password: 'securePassword123!',
);

// Login
await supabase.auth.signInWithPassword(
  email: 'user@email.com',
  password: 'securePassword123!',
);

// Password reset
await supabase.auth.resetPasswordForEmail('user@email.com');
```

---

### Magic Link / OTP Email
> ⚠️ Requires deep link configuration. See `setup.md` for `supabase.auth.setExternalAuthAction` and URI scheme config.

```dart
// Send OTP/Magic Link
await supabase.auth.signInWithOtp(email: 'user@email.com');

// Verify OTP (from email code entry)
await supabase.auth.verifyOTP(
  email: 'user@email.com',
  token: '123456',
  type: OtpType.email,
);
```

---

### Phone OTP
```dart
// Send OTP
await supabase.auth.signInWithOtp(phone: '+886912345678');

// Verify OTP
await supabase.auth.verifyOTP(
  phone: '+886912345678',
  token: userInputCode,
  type: OtpType.sms,
);
```

---

### Native Google Sign-In
Supabase requires the **Web Client ID** (from Google Cloud Console) as `serverClientId` — this is how Supabase's server decrypts the `idToken`.

> Uses `google_sign_in ^7.2.0` stream-based API.

```dart
import 'package:google_sign_in/google_sign_in.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

Future<AuthResponse?> nativeGoogleSignIn() async {
  await GoogleSignIn.instance.initialize(
    serverClientId: 'YOUR_WEB_CLIENT_ID.apps.googleusercontent.com',
  );

  // Attempt silent re-sign-in first (returning users)
  GoogleSignIn.instance.attemptLightweightAuthentication();

  // For cold sign-in, call authenticate() from a button press
  await GoogleSignIn.instance.authenticate();

  final account = GoogleSignIn.instance.currentUser;
  if (account == null) return null;

  final serverAuth = await account.authorizationClient.authorizeServer([]);
  if (serverAuth?.serverAuthCode == null) return null;

  // Exchange tokens with Supabase
  final auth = await account.authorizationClient.authorizationForScopes([]);
  if (auth?.idToken == null) throw const AuthException('No ID token');

  return supabase.auth.signInWithIdToken(
    provider: OAuthProvider.google,
    idToken: auth!.idToken!,
    accessToken: auth.accessToken,
  );
}
```

---

### Native Apple Sign-In
Supabase requires a `nonce` (SHA-256 hashed) to prevent replay attacks. The **raw nonce** is passed to Supabase; the **hashed nonce** is passed to Apple.

```dart
import 'dart:convert';
import 'dart:math';
import 'package:crypto/crypto.dart';
import 'package:sign_in_with_apple/sign_in_with_apple.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

String _generateNonce([int length = 32]) {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._';
  final random = Random.secure();
  return List.generate(length, (_) => chars[random.nextInt(chars.length)]).join();
}

Future<AuthResponse> signInWithApple() async {
  final rawNonce = _generateNonce();
  final hashedNonce = sha256.convert(utf8.encode(rawNonce)).toString();

  final credential = await SignInWithApple.getAppleIDCredential(
    scopes: [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
    nonce: hashedNonce, // hashed nonce → Apple
  );

  final idToken = credential.identityToken;
  if (idToken == null) throw const AuthException('No ID Token from Apple');

  final response = await supabase.auth.signInWithIdToken(
    provider: OAuthProvider.apple,
    idToken: idToken,
    nonce: rawNonce, // raw nonce → Supabase
  );

  // ⚠️ Apple only provides name on FIRST login — persist it immediately
  if (credential.givenName != null) {
    await supabase.auth.updateUser(
      UserAttributes(data: {
        'full_name': '${credential.givenName} ${credential.familyName}',
        'given_name': credential.givenName,
        'family_name': credential.familyName,
      }),
    );
  }

  return response;
}
```

---

### Sign-Out
```dart
await supabase.auth.signOut();
// onAuthStateChange emits AuthChangeEvent.signedOut → navigate to Login
```

---

## Constraints
* **Web Client ID is mandatory for Google**: The iOS/Android Client ID alone is NOT sufficient for Supabase. You MUST use the Web Client ID as `serverClientId`.
* **Deep Link Setup for OTP/Magic Link**: `signInWithOtp` only works if your app handles the return URL redirect. Configure `io.supabase.yourapp://login-callback` in Supabase Dashboard → Auth → URL Configuration.
* **Apple name is one-time**: Persist `givenName` + `familyName` via `updateUser(..)` immediately after first sign-in.
* **Nonce is mandatory for Apple**: Apple's `idToken` without a matching nonce will fail server validation in Supabase.
