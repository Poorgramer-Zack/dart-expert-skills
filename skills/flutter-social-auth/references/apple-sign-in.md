---
name: "flutter-apple-sign-in"
description: "Best practices for implementing Sign in with Apple using sign_in_with_apple v7.x. Includes Apple Developer Portal setup (App ID, Service ID, Auth Keys), iOS/Android/Web configuration, server-side token validation, minimal viable examples, comprehensive error handling patterns, and critical edge cases."
metadata:
  last_modified: "2026-03-31 20:19:44 (GMT+8)"
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

---

## 🎯 Minimal Viable Example

### Complete Working Implementation
```dart
import 'package:flutter/material.dart';
import 'package:sign_in_with_apple/sign_in_with_apple.dart';

class AppleAuthService {
  String? _userIdentifier;
  
  // Sign-in with comprehensive error handling
  Future<AppleSignInResult> signIn() async {
    try {
      final credential = await SignInWithApple.getAppleIDCredential(
        scopes: [
          AppleIDAuthorizationScopes.email,
          AppleIDAuthorizationScopes.fullName,
        ],
      );

      // ⚠️ CRITICAL: Save name/email NOW - only provided once!
      if (credential.givenName != null || credential.email != null) {
        await _saveUserInfo(
          email: credential.email,
          firstName: credential.givenName,
          lastName: credential.familyName,
        );
      }

      _userIdentifier = credential.userIdentifier;

      // Send auth code to server for validation (expires in 5 minutes!)
      await _sendToServer(credential.authorizationCode);

      return AppleSignInResult.success(credential);
      
    } on SignInWithAppleAuthorizationException catch (e) {
      return _handleAppleException(e);
    } on Exception catch (e) {
      return AppleSignInResult.error('Unexpected error: $e');
    }
  }

  // Check if session is still valid (iOS only)
  Future<bool> isSessionValid() async {
    if (_userIdentifier == null) return false;
    
    try {
      final state = await SignInWithApple.getCredentialState(_userIdentifier!);
      return state == AppleIDAuthorizationCredentialState.authorized;
    } catch (e) {
      return false;
    }
  }

  // Sign-in for Android/Web (requires Service ID)
  Future<AppleSignInResult> signInCrossPlatform({
    required String serviceId,
    required Uri redirectUri,
  }) async {
    try {
      final credential = await SignInWithApple.getAppleIDCredential(
        scopes: [
          AppleIDAuthorizationScopes.email,
          AppleIDAuthorizationScopes.fullName,
        ],
        webAuthenticationOptions: WebAuthenticationOptions(
          clientId: serviceId,
          redirectUri: redirectUri,
        ),
      );

      return AppleSignInResult.success(credential);
      
    } on SignInWithAppleAuthorizationException catch (e) {
      return _handleAppleException(e);
    }
  }

  // Handle Apple-specific exceptions
  AppleSignInResult _handleAppleException(
    SignInWithAppleAuthorizationException e,
  ) {
    switch (e.code) {
      case AuthorizationErrorCode.canceled:
        return AppleSignInResult.canceled();
      case AuthorizationErrorCode.failed:
        return AppleSignInResult.error('Sign-in failed: ${e.message}');
      case AuthorizationErrorCode.invalidResponse:
        return AppleSignInResult.error('Invalid response from Apple');
      case AuthorizationErrorCode.notHandled:
        return AppleSignInResult.error('Request not handled');
      case AuthorizationErrorCode.notInteractive:
        return AppleSignInResult.error('Not in interactive context');
      case AuthorizationErrorCode.unknown:
      default:
        return AppleSignInResult.error('Unknown error: ${e.message}');
    }
  }

  Future<void> _saveUserInfo({
    String? email,
    String? firstName,
    String? lastName,
  }) async {
    // Save to secure storage or database
    // This is CRITICAL - Apple only provides this once!
  }

  Future<void> _sendToServer(String authCode) async {
    // Send authorization code to backend within 5 minutes
  }
}

// Result class for type-safe error handling
class AppleSignInResult {
  final AuthorizationCredentialAppleID? credential;
  final String? error;
  final bool isCanceled;

  AppleSignInResult._({this.credential, this.error, this.isCanceled = false});

  factory AppleSignInResult.success(AuthorizationCredentialAppleID credential) {
    return AppleSignInResult._(credential: credential);
  }

  factory AppleSignInResult.canceled() {
    return AppleSignInResult._(isCanceled: true);
  }

  factory AppleSignInResult.error(String message) {
    return AppleSignInResult._(error: message);
  }

  bool get isSuccess => credential != null;
}

// UI Widget Example
class AppleSignInButton extends StatelessWidget {
  final AppleAuthService authService;

  const AppleSignInButton({required this.authService});

  @override
  Widget build(BuildContext context) {
    return SignInWithAppleButton(
      onPressed: () async {
        final result = await authService.signIn();
        
        if (result.isSuccess) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text('Signed in with Apple')),
          );
        } else if (!result.isCanceled && result.error != null) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text('Error: ${result.error}')),
          );
        }
      },
    );
  }
}
```

---

## 🚨 Error Handling Patterns

### Common Error Codes & Solutions

| Error Code | Description | Solution |
|------------|-------------|----------|
| `canceled` | User dismissed the sign-in dialog | Handle gracefully, no action needed |
| `failed` | Generic failure (network, config) | Check Service ID, redirect URI, network |
| `invalidResponse` | Malformed response from Apple | Retry or contact Apple Developer Support |
| `notHandled` | Request wasn't processed | Check platform configuration |
| `unknown` | Unspecified error | Log details, implement fallback |

### Platform-Specific Error Handler
```dart
class AppleSignInErrorHandler {
  static String getUserMessage(SignInWithAppleAuthorizationException error) {
    switch (error.code) {
      case AuthorizationErrorCode.canceled:
        return 'Sign-in was canceled';
      case AuthorizationErrorCode.failed:
        return 'Sign-in failed. Please try again.';
      case AuthorizationErrorCode.invalidResponse:
        return 'Invalid response. Please try again later.';
      case AuthorizationErrorCode.notHandled:
        return 'Request not processed. Check your connection.';
      case AuthorizationErrorCode.notInteractive:
        return 'Cannot sign in at this time';
      default:
        return 'An error occurred: ${error.message ?? 'Unknown'}';
    }
  }

  static bool shouldRetry(AuthorizationErrorCode code) {
    return code == AuthorizationErrorCode.failed ||
           code == AuthorizationErrorCode.notHandled;
  }

  static bool isConfigurationError(AuthorizationErrorCode code) {
    return code == AuthorizationErrorCode.invalidResponse;
  }
}
```

### Server-Side Token Validation
```dart
// After receiving auth code on server, validate it
class AppleTokenValidator {
  static const String tokenEndpoint = 'https://appleid.apple.com/auth/token';
  static const String keysEndpoint = 'https://appleid.apple.com/auth/keys';
  
  Future<bool> validateAuthCode({
    required String authCode,
    required String clientId,  // Use Bundle ID for iOS, Service ID for web/Android
    required String clientSecret,  // JWT signed with your .p8 key
  }) async {
    try {
      final response = await http.post(
        Uri.parse(tokenEndpoint),
        body: {
          'client_id': clientId,
          'client_secret': clientSecret,
          'code': authCode,
          'grant_type': 'authorization_code',
        },
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body);
        // Verify identity_token JWT signature against Apple's public keys
        await _verifyIdentityToken(data['id_token']);
        return true;
      }
      
      return false;
    } catch (e) {
      print('Token validation failed: $e');
      return false;
    }
  }

  Future<void> _verifyIdentityToken(String idToken) async {
    // 1. Decode JWT header to get 'kid' (key ID)
    // 2. Fetch Apple's public keys from keysEndpoint
    // 3. Verify signature using matching public key
    // 4. Verify claims (iss, aud, exp)
  }
}
```

---

## ⚠️ Common Gotchas

### 1. **Name/Email Only Provided ONCE (Most Critical)**
**Cause**: Apple only sends `givenName`, `familyName`, and `email` on the **first** authorization.

**Solution**:
```dart
// ALWAYS save immediately on first sign-in
final credential = await SignInWithApple.getAppleIDCredential(...);

if (credential.givenName != null || credential.email != null) {
  // This is the ONLY time you'll get this data!
  await database.saveUser(
    id: credential.userIdentifier,
    email: credential.email,  // May be null if user chose not to share
    firstName: credential.givenName,
    lastName: credential.familyName,
  );
}

// On subsequent logins, these fields will be NULL
// You must read from your database instead
```

### 2. **Authorization Code Expires in 5 Minutes**
**Cause**: The `authorizationCode` is single-use and short-lived.

**Solution**:
```dart
final credential = await SignInWithApple.getAppleIDCredential(...);

// Send to server IMMEDIATELY - don't wait!
try {
  await http.post(
    Uri.parse('https://yourapi.com/apple-auth'),
    body: {'code': credential.authorizationCode},
  ).timeout(Duration(seconds: 30));  // Fail fast if server is slow
} catch (e) {
  // Auth code might be expired now - must re-authenticate
  showError('Session expired. Please sign in again.');
}
```

### 3. **Simulator vs. Real Device**
**Cause**: Sign in with Apple requires iCloud-signed-in account.

**Solution**:
- **iOS 13.5+ Simulator**: May work with signed-in iCloud account
- **Production**: ALWAYS test on real device before release
- Check Settings → Apple ID to ensure user is signed in

### 4. **Android Deep Link Fails**
**Cause**: Incorrect `launchMode` in `AndroidManifest.xml`.

**Solution**:
```xml
<activity
  android:name=".MainActivity"
  android:launchMode="singleTop">  <!-- NOT singleTask! -->
  <!-- Chrome Custom Tab needs to return to existing activity -->
</activity>
```

### 5. **Service ID Redirect URI Mismatch**
**Cause**: URL registered in Apple Developer Portal doesn't match code.

**Solution**:
```dart
// These MUST match exactly (including https://)
final redirectUri = Uri.parse('https://example.com/callbacks/apple');

// In Apple Developer Portal → Service ID → Configure:
// ✅ Return URL: https://example.com/callbacks/apple
// ❌ Return URL: https://example.com/callbacks/apple/ (trailing slash)
// ❌ Return URL: http://example.com/callbacks/apple (http, not https)
```

### 6. **Email is Null Even with Permission**
**Cause**: User chose "Hide My Email" or didn't share email.

**Solution**:
```dart
final credential = await SignInWithApple.getAppleIDCredential(...);

if (credential.email == null) {
  // User chose not to share email
  // Option 1: Ask for email manually
  final email = await showEmailInputDialog();
  
  // Option 2: Use userIdentifier as unique ID (no email required)
  await createAccount(userId: credential.userIdentifier);
}
```

### 7. **Missing .p8 Key on Server**
**Cause**: Auth key not downloaded or lost.

**Solution**:
- Download `.p8` file **immediately** after creating key (can only download once)
- If lost, create a new key in Apple Developer Portal
- Store securely (server-side only, never in app)

### 8. **Wrong Client ID on Server**
**Cause**: Using Service ID for iOS native clients.

**Solution**:
```dart
// Server-side validation logic
String getClientId(String platform) {
  switch (platform) {
    case 'ios':
    case 'macos':
      return 'com.example.app';  // Bundle ID (App ID)
    case 'android':
    case 'web':
      return 'com.example.app.service';  // Service ID
    default:
      throw ArgumentError('Unknown platform: $platform');
  }
}
```

### 9. **Session Revoked Without Notice**
**Cause**: User revoked access via Apple Settings.

**Solution**:
```dart
// Check credential state periodically (iOS only)
Future<void> checkSessionHealth() async {
  if (_userIdentifier == null) return;
  
  final state = await SignInWithApple.getCredentialState(_userIdentifier!);
  
  if (state == AppleIDAuthorizationCredentialState.revoked) {
    // User revoked access - clear session and re-authenticate
    await logout();
    showMessage('Session expired. Please sign in again.');
  }
}

// Run this on app launch or periodically
```

---

## Constraints
* **Apple Developer Program**: Required (paid subscription). Free accounts cannot use Sign in with Apple.
* **`authorizationCode` is single-use**: Must be exchanged with Apple within **5 minutes**. Pass it to your server immediately.
* **Server daily re-verification**: This is mandatory per Apple's guidelines. Failure to implement it risks unexpected session revocations.
* **`identityToken` validation**: Always verify the JWT signature on the server — never trust it blindly on the client.
