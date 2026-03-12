---
name: "serverpod-auth-and-social-sign-in"
description: "Detailed guide for Serverpod authentication system, covering core setup, social identity providers (Google, Apple, Firebase, Facebook), and Distributed Redis caching."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Serverpod Authentication & Social Sign-In

## Goal
Implement a production-ready authentication system in Serverpod. This includes setting up the built-in `serverpod_auth` module, configuring social identity providers (Google, Apple, Firebase, Facebook), and ensuring secure session management via JWT or Server-side sessions.

---

## 🚀 Server-Side Setup

### 1. Installation
Add the auth module to your server's `pubspec.yaml`:
```yaml
dependencies:
  serverpod_auth_idp_server: ^2.3.0 # Match your Serverpod version
```

### 2. Token Manager — Session vs JWT
Choose one (or both) token strategies:

| Strategy | Config Key | Trade-off |
|---|---|---|
| **JWT** | `JwtConfigFromPasswords` | Stateless; no DB lookup per request. Tokens are self-contained. |
| **Server-Side Session** | `ServerSideSessionsConfigFromPasswords` | Revocable at any time from server. Requires DB lookup per request. |

For most production apps, **JWT** is recommended. Use Server-Side Sessions only if you need instant revocation.

### 3. Configure Authentication Services (`server.dart`)
Initialize authentication services with a Token Manager and Identity Providers.

```dart
import 'package:serverpod/serverpod.dart';
import 'package:serverpod_auth_idp_server/core.dart';
import 'package:serverpod_auth_idp_server/providers/google.dart';
import 'package:serverpod_auth_idp_server/providers/apple.dart';
import 'package:serverpod_auth_idp_server/providers/facebook.dart';

void run(List<String> args) async {
  final pod = Serverpod(args, Protocol(), Endpoints());

  pod.initializeAuthServices(
    tokenManagerBuilders: [
      JwtConfigFromPasswords(), // Reads from passwords.yaml or env vars
    ],
    identityProviderBuilders: [
      EmailIdpConfigFromPasswords(),
      GoogleIdpConfigFromPasswords(),
      AppleIdpConfigFromPasswords(),
      FacebookIdpConfigFromPasswords(),
      // PasskeyIdpConfigFromPasswords(), // Experimental: FIDO2 Passkey
    ],
  );

  // Required for Apple Sign-In web/Android callback routes
  pod.configureAppleIdpRoutes();

  await pod.start();
}
```

### 3. Expose Endpoints
You MUST subclass the base endpoints for each provider to expose them to the client.

```dart
// endpoints/auth_endpoints.dart
import 'package:serverpod_auth_idp_server/core.dart' as core;
import 'package:serverpod_auth_idp_server/providers/email.dart' as email;
import 'package:serverpod_auth_idp_server/providers/google.dart' as google;
import 'package:serverpod_auth_idp_server/providers/apple.dart' as apple;
import 'package:serverpod_auth_idp_server/providers/facebook.dart' as facebook;

class RefreshJwtTokensEndpoint extends core.RefreshJwtTokensEndpoint {}
class EmailIdpEndpoint extends email.EmailIdpBaseEndpoint {}
class GoogleIdpEndpoint extends google.GoogleIdpBaseEndpoint {}
class AppleIdpEndpoint extends apple.AppleIdpBaseEndpoint {}
class FacebookIdpEndpoint extends facebook.FacebookIdpBaseEndpoint {}
```

---

## 🔑 Provider Specifics

### `config/passwords.yaml` Key Reference
All secrets must be defined here (or as `SERVERPOD_PASSWORD_<key>=value` environment variables). **Never commit this file to version control.**

| Key | Required By | Description |
|---|---|---|
| `serverSideSessionKeyHashPepper` | ServerSideSessions | Random string for session key hashing |
| `jwtRefreshTokenHashPepper` | JWT | Random string for refresh token hashing |
| `jwtHmacSha512PrivateKey` | JWT | HMAC-SHA512 private key for signing JWTs |
| `emailSecretHashPepper` | Email IDP | Pepper for email verification token hashing |
| `googleClientSecret` | Google IDP | Full Google OAuth2 Web client JSON (paste entire JSON) |
| `appleKey` | Apple IDP | Contents of the `.p8` private key file |
| `appleTeamId` | Apple IDP | 10-character Apple Team ID |
| `appleKeyId` | Apple IDP | 10-character Key ID from Apple Developer Portal |
| `appleServiceIdentifier` | Apple IDP | Service ID identifier (for web/Android callback) |
| `appleBundleIdentifier` | Apple IDP | Your iOS Bundle ID |
| `facebookAppId` | Facebook IDP | Facebook App ID from Meta Console |
| `facebookAppSecret` | Facebook IDP | Facebook App Secret from Meta Console |
| `firebaseServiceAccountKey` | Firebase bridge | Full Firebase service account JSON |

```yaml
# config/passwords.yaml (NEVER commit this file)
development:
  serverSideSessionKeyHashPepper: 'random-string-here'
  jwtRefreshTokenHashPepper: 'random-string-here'
  jwtHmacSha512PrivateKey: 'your-private-key'
  emailSecretHashPepper: 'random-string-here'
  googleClientSecret: '{"type":"service_account",...}'
  appleKey: '-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----'
  appleTeamId: 'ABCDE12345'
  appleKeyId: 'FGHIJ67890'
  facebookAppId: '1234567890'
  facebookAppSecret: 'your-app-secret'
```

### Google Sign-In
- **Credentials**: Create a **Web application** OAuth client in Google Cloud Console. Download JSON.
- **passwords.yaml**: Paste the entire JSON content into `googleClientSecret`.
- **Web**: Add `<meta name="google-signin-client_id" content="YOUR_CLIENT_ID">` to `index.html`.

### Apple Sign-In
- **Prerequisites**: Apple Developer Program account. See `flutter-social-auth/references/apple-sign-in.md` for full portal setup.
- **Android/Web Support**: Configure `redirectUri` pointing to `https://yourdomain.com/auth/callback`.
- **Required**: `pod.configureAppleIdpRoutes()` must be called in `server.dart`.

### Facebook Sign-In
- **Client Package**: Requires `serverpod_auth_idp_flutter_facebook` (separate from core client).
- **Web/macOS**: MUST call `initializeFacebookSignIn(appId: '...')` before login.
- **Permissions**: Default is `['email', 'public_profile']`.

### Firebase Auth Integration
- **Role**: Serverpod verifies Firebase `ID Token`; Firebase handles the auth UX.
- **Config**: `firebaseServiceAccountKey` must be the full service account JSON.
- **Flutter**: Use `FirebaseAuthController.login(user)` to sync Firebase user to Serverpod session.

---

## 📱 Flutter Client Setup

### 1. Initialize `FlutterAuthSessionManager`
Create the client and initialize the session manager in `main()`.

```dart
late Client client;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  client = Client('http://localhost:8080/')
    ..connectivityMonitor = FlutterConnectivityMonitor()
    ..authSessionManager = FlutterAuthSessionManager();

  // Restore session and initialize social services
  await client.auth.initialize();
  client.auth.initializeGoogleSignIn();
  client.auth.initializeAppleSignIn();
  client.auth.initializeFacebookSignIn(appId: 'YOUR_FB_APP_ID'); // appId required for Web/macOS

  runApp(const MyApp());
}
```

### 2. Using `SignInWidget`
The easiest way to present social login buttons. It automatically detects enabled providers on the server.

```dart
SignInWidget(
  client: client,
  onAuthenticated: () {
    // Session is now active. Use ref.invalidate or similar to update UI.
  },
  onError: (e) => print('Auth Error: $e'),
)
```

---

## ⚡ Distributed Caching (Redis)
For high-concurrency, use Redis to store frequently accessed Yaml-defined objects.

```dart
Future<UserData> getUser(Session session, int id) async {
  final key = 'user_$id';
  
  // 1. Check Redis (Global Cache)
  var user = await session.caches.global.get<UserData>(key);
  if (user != null) return user;

  // 2. Fallback to DB
  user = await UserData.db.findById(session, id);
  if (user != null) {
    await session.caches.global.put(key, user, lifetime: Duration(minutes: 5));
  }
  return user!;
}
```

## 🛠️ Migration Command Sequence
Run these in order each time you add or change the auth schema:
```bash
# 1. Regenerate client code and endpoint stubs
serverpod generate

# 2. Create the migration SQL
serverpod create-migration

# 3. Start the database container
docker compose up --build --detach

# 4. Apply the migration (maintenance mode)
dart run bin/main.dart --role maintenance --apply-migrations
```
> **Warning**: Skipping `serverpod create-migration` after adding a new provider will result in missing database tables and runtime errors.

---

## Constraints
* **Passwords Hygiene**: NEVER commit `config/passwords.yaml`. Use `SERVERPOD_PASSWORD_<key>=value` environment variables for production.
* **Migration Mandatory**: After adding `serverpod_auth`, you MUST run the full migration sequence above to create system tables.
* **Apple Callback**: `pod.configureAppleIdpRoutes()` must be called in `server.dart` for web/Android callback routes to function.
* **Facebook Separate Package**: Facebook login on Flutter requires `serverpod_auth_idp_flutter_facebook` — NOT included in the core auth client.
* **JWT Key Length**: `jwtHmacSha512PrivateKey` should be a long (≥64 chars) cryptographically random string.
* **Passkey is Experimental**: `PasskeyIdpConfigFromPasswords` is available but not production-ready as of Serverpod 2.x.
