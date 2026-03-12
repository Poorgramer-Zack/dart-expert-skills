---
name: "flutter-google-sign-in"
description: "Best practices for implementing Google Sign-In in Flutter apps using google_sign_in v7.x. Includes stream-based auth flow, Android/iOS/Web configuration, authorization scopes, and server auth code exchange."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Google Sign-In Best Practices

## Goal
Implement a reliable, cross-platform Google Sign-In using `google_sign_in ^7.2.0`. The v7 release introduced a breaking redesign: authentication is now stream-based rather than a single `signIn()` call. This doc covers the new API, platform setup, and common pitfalls.

## Process

### 1. Installation
```yaml
dependencies:
  google_sign_in: ^7.2.0
  firebase_auth: ^5.4.0 # Optional but recommended for backend session
```

### 2. Provider Configuration

#### Android
*   **SHA Fingerprints**: Register both **SHA-1** (debug & release) and **SHA-256** in Firebase Console or Google Cloud Console.
    - Extract debug SHA-1: `./gradlew signingReport`
    - For Google Play App Signing: get fingerprint from Play Console → use it directly (not local keystore).
*   **google-services.json**: Download from Firebase Console and place at `android/app/`.
*   **OAuth Consent Screen**: Fill out **every** field (even optional) in Google Cloud Console. Incomplete consent screens cause `ApiException: 10`.

#### iOS & macOS
*   **GoogleService-Info.plist**: Download from Firebase Console and add to Xcode project (Runner target).
*   **Info.plist**: Add the `REVERSED_CLIENT_ID` from `GoogleService-Info.plist` as a URL scheme.
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>com.googleusercontent.apps.YOUR_IOS_CLIENT_ID</string>
    </array>
  </dict>
</array>
```

#### Web
*   **index.html**: Add the client ID meta tag inside `<head>`.
```html
<meta name="google-signin-client_id" content="YOUR_WEB_CLIENT_ID.apps.googleusercontent.com">
```
*   **Web Token Expiry**: Access tokens expire after **3600 seconds**. Your app must detect failed API calls and prompt the user to re-authorize.

---

### 3. Implementation (v7 Stream-based API)

#### v7 Breaking Change Summary
| v6.x (Deprecated) | v7.x (Current) |
|---|---|
| `GoogleSignIn()` constructor | `GoogleSignIn.instance` singleton |
| `signIn()` Future | `authenticate()` + `authenticationEvents` Stream |
| `signOut()` | `signOut()` (unchanged) |
| `currentUser` | Listen to `authenticationEvents` |

#### Initialization in `main.dart`
```dart
import 'package:google_sign_in/google_sign_in.dart';

final _signIn = GoogleSignIn.instance;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize and listen for authentication state changes
  unawaited(
    _signIn.initialize(
      // clientId: required on Web/Android, read from config on iOS
      // serverClientId: required if you need a server auth code
    ).then((_) {
      _signIn.authenticationEvents
          .listen(_handleAuthEvent)
          .onError(_handleAuthError);

      // Silently re-sign-in returning users
      _signIn.attemptLightweightAuthentication();
    }),
  );

  runApp(const MyApp());
}

void _handleAuthEvent(GoogleSignInAuthenticationEvent event) {
  if (event is GoogleSignInAuthenticationEventSignIn) {
    final account = event.user;
    // Update Riverpod/Provider state with account.email, account.displayName
  } else if (event is GoogleSignInAuthenticationEventSignOut) {
    // Clear session state
  }
}
```

#### Initiating Sign-In
Use `supportsAuthenticate()` to determine the correct UI approach (native dialog vs. web SDK button).

```dart
Widget buildSignInButton() {
  if (GoogleSignIn.instance.supportsAuthenticate()) {
    return ElevatedButton(
      onPressed: () async {
        try {
          await GoogleSignIn.instance.authenticate();
        } catch (e) {
          // Handle errors
        }
      },
      child: const Text('Sign in with Google'),
    );
  } else {
    // On web, must use the SDK-rendered button
    return GoogleSignInWebButton(); // from google_sign_in_web
  }
}
```

#### Sign-Out
```dart
await GoogleSignIn.instance.signOut();
```

---

### 4. Authorization Scopes (Beyond Identity)
Authentication provides user identity. To call Google APIs on behalf of the user, you must request authorization separately.

```dart
const scopes = ['https://www.googleapis.com/auth/contacts.readonly'];

// Check if already authorized (silent)
final GoogleSignInClientAuthorization? auth = await account
    .authorizationClient
    .authorizationForScopes(scopes);

// If not authorized, request from user (must be in a button press on some platforms)
if (auth == null) {
  final newAuth = await account.authorizationClient.authorizeScopes(scopes);
  // Use newAuth.accessToken for API calls
}
```

#### Requesting a Server Auth Code
If your backend needs to access Google APIs on behalf of the user:
```dart
final serverAuth = await account.authorizationClient.authorizeServer(scopes);
// Send serverAuth.serverAuthCode to your server
// Server exchanges it for access + refresh tokens
```
> **Warning**: Server auth codes may only be available on initial sign-in on some platforms. Request them immediately after sign-in and manage tokens server-side.

---

## 🛠️ Troubleshooting (FAQ)

### Q: `ApiException: 10` on Android?
**A:** Two most common causes:
1. **SHA mismatch**: Run `./gradlew signingReport`, compare with what's in Firebase/Google Console. Both debug and release fingerprints must be registered.
2. **Incomplete OAuth Consent Screen**: Fill in **all** fields including Support Email, App Logo, and App Domain — even if marked optional.

### Q: Web token stops working after an hour?
**A:** Web `accessToken` expires in 3600s and is NOT auto-refreshed. Detect the 401 error from your API call and call `authorizeScopes()` again to get a fresh token.

### Q: Silent login doesn't restore session on cold start?
**A:** Ensure `attemptLightweightAuthentication()` is called after `initialize()` completes. This triggers `authenticationEvents` with the cached user without showing any UI.

---

## Constraints
* **v7 Migration**: Do NOT use the old `signIn()` / `currentUser` pattern — they are removed in v7.
* **Scope Minimization**: Request only the scopes you need (least privilege).
* **Authorization Requires User Interaction**: On platforms where `authorizationRequiresUserInteraction()` returns `true`, scope requests MUST be triggered by a user action (e.g., button press), not programmatically on app start.
