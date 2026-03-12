---
name: "flutter-apple-sign-in"
description: "Best practices for implementing Sign in with Apple using sign_in_with_apple v7.x. Includes Apple Developer Portal setup (App ID, Service ID, Auth Keys), iOS/Android/Web configuration, server-side token validation, and critical edge cases."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Apple Sign-In Best Practices

## Goal
Implement a compliant, cross-platform Sign in with Apple using `sign_in_with_apple ^7.0.1`. This guide covers the complete Apple Developer Portal setup, platform-specific native configuration, and the mandatory server-side authorization code validation that many guides omit.

> **App Store Policy**: If your iOS/macOS app offers any other 3rd-party social login (Google, Facebook, etc.), Sign in with Apple is **mandatory** to pass App Store review.

---

## Process

### Installation
```yaml
dependencies:
  sign_in_with_apple: ^7.0.1
```

### Apple Developer Portal Setup

#### Step A: Register App ID (iOS/macOS)
1. Go to [Identifiers → App IDs](https://developer.apple.com/account/resources/identifiers/list/bundleId)
2. Click **Register an App ID** → Select **App IDs** → Continue
3. Set **Description** and **Bundle ID** (e.g. `com.example.myapp`)
4. Enable the **Sign in with Apple** capability → Continue → Register
5. If using an existing App ID: open it from the list → check **Sign in with Apple** → Save

#### Step B: Create Service ID (Android & Web ONLY)
The Service ID acts as the `clientId` for the web OAuth flow.

1. Go to [Identifiers → Service IDs](https://developer.apple.com/account/resources/identifiers/list/serviceId)
2. Click **Register a Services ID** → Select **Services IDs** → Continue
3. Set **Description** and **Identifier** (this Identifier = your `clientId`)
4. Click **Configure** next to Sign in with Apple:
   - **Domains and Subdomains**: Add your server's domain (e.g. `example.com`)
   - **Return URLs**: Add the full callback URL (e.g. `https://example.com/callbacks/sign_in_with_apple`)
5. Click Done → Continue → Save

#### Step C: Create Auth Key (One-Time Download!)
Used by your **server** to validate authorization codes with Apple.

1. Go to [Keys](https://developer.apple.com/account/resources/authkeys/list)
2. Click **Create a key** → set Key Name → check **Sign in with Apple** → Configure
3. Under **Primary App ID**, select your App ID → Save → Continue → Register
4. **Download the `.p8` key file immediately** — it can only be downloaded ONCE
5. Note the **Key ID** for your server configuration

---

### Platform Configuration

#### iOS (Xcode)
1. Open `ios/Runner.xcworkspace` in Xcode
2. Select **Runner target** → **Signing & Capabilities**
3. Click **+ Capability** → add **Sign in with Apple**
4. If not using auto-signing, re-download provisioning profiles

#### Android (`AndroidManifest.xml`)
Apple Sign-In on Android uses a web-based OAuth flow via Chrome Custom Tabs.
```xml
<!-- In <activity android:name=".MainActivity" ...> -->
<activity
  android:name=".MainActivity"
  android:launchMode="singleTop">
  <!-- singleTop: Chrome Custom Tab survives both app switcher and home screen launch -->
  <!-- singleTask: dismissed when launched from home screen icon  -->
</activity>
```
> **Critical**: Must use `launchMode="singleTop"` or `singleTask` — otherwise deep link from Chrome Custom Tab back to app will fail.

---

### Implementation

#### Basic iOS Flow
```dart
import 'package:sign_in_with_apple/sign_in_with_apple.dart';

Future<AuthorizationCredentialAppleID?> signIn() async {
  try {
    final credential = await SignInWithApple.getAppleIDCredential(
      scopes: [
        AppleIDAuthorizationScopes.email,
        AppleIDAuthorizationScopes.fullName,
      ],
    );

    // credential.authorizationCode → send to server for validation
    // credential.identityToken → JWT, can be decoded for user info
    // credential.userIdentifier → stable unique user ID
    // ⚠️ credential.email / credential.givenName: ONLY populated on FIRST login
    return credential;
  } on SignInWithAppleAuthorizationException catch (e) {
    if (e.code == AuthorizationErrorCode.canceled) return null;
    rethrow;
  }
}
```

#### Android & Web (Service ID + Redirect URI)
```dart
final credential = await SignInWithApple.getAppleIDCredential(
  scopes: [
    AppleIDAuthorizationScopes.email,
    AppleIDAuthorizationScopes.fullName,
  ],
  webAuthenticationOptions: WebAuthenticationOptions(
    clientId: 'com.example.myapp.service', // Service ID identifier
    redirectUri: Uri.parse('https://example.com/callbacks/sign_in_with_apple'),
  ),
);
```

#### Session Validity Check (iOS)
```dart
Future<bool> isSessionValid(String userIdentifier) async {
  final state = await SignInWithApple.getCredentialState(userIdentifier);
  return state == AppleIDAuthorizationCredentialState.authorized;
}
```

---

### Server-Side Validation (Required!)
After receiving `authorizationCode` from the client, your server must:

1. Exchange the `authorizationCode` for an `access_token` + `refresh_token` using Apple's servers
2. Validate the `identityToken` (JWT) against Apple's public keys
3. **Re-verify daily** via the `refresh_token` — if Apple returns `invalid_grant`, the user has revoked access and the session must be revoked in your system as well
4. **Client ID on server**: Use the **App ID** (Bundle ID) for native iOS/macOS clients; use the **Service ID** for web/Android clients

---

## 🛠️ Troubleshooting (FAQ)

### Q: Why do I only get name/email once?
**A:** Apple only provides `givenName`, `familyName`, and `email` on the **very first** authorization. You MUST persist these to your database immediately. Subsequent logins return `null` for these fields.

### Q: Sign-In works on device but fails on simulator?
**A:** Sign in with Apple requires a physical device with an iCloud-signed-in account. Some simulators may work on specific iOS versions (13.5+), but real device testing is mandatory for production verification.

### Q: Android deep link doesn't return to app?
**A:** Verify `launchMode` is set to `singleTop` or `singleTask` in `AndroidManifest.xml`. Also check that the `redirectUri` in your Service ID config matches exactly what you pass in code.

---

## Constraints
* **Apple Developer Program**: Required (paid subscription). Free accounts cannot use Sign in with Apple.
* **`authorizationCode` is single-use**: Must be exchanged with Apple within **5 minutes**. Pass it to your server immediately.
* **Server daily re-verification**: This is mandatory per Apple's guidelines. Failure to implement it risks unexpected session revocations.
* **`identityToken` validation**: Always verify the JWT signature on the server — never trust it blindly on the client.
