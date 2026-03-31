---
name: "flutter-facebook-login"
description: "Best practices for implementing Facebook Login using flutter_facebook_auth v7.x. Includes Android/iOS/Web configuration, iOS 17 App Tracking Transparency handling, Limited Login mode, multi-provider Info.plist merging, minimal viable examples, comprehensive error handling patterns, and common gotchas."
metadata:
  last_modified: "2026-03-31 20:19:44 (GMT+8)"
---

# Facebook Login Best Practices

## Goal
Implement a reliable, cross-platform Facebook Login using `flutter_facebook_auth ^7.1.5`. This guide focuses on the critical native configuration steps, iOS 17 privacy requirements, the Limited Login mode behavioral changes, and correct co-existence with other providers.

> **Warning**: The Facebook SDK on Android throws an Exception immediately on startup if `strings.xml` is not configured. This blocks ALL other plugins. Configure Android before running the project.

---

## Process

### 1. Installation
```yaml
dependencies:
  flutter_facebook_auth: ^7.1.5
```
> **Swift only**: The iOS plugin is written in Swift. Objective-C Flutter projects are **not supported**. Ensure your Flutter project uses Swift as the default iOS language.

---

### 2. Android Configuration

#### `android/app/src/main/res/values/strings.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="facebook_app_id">YOUR_APP_ID</string>
    <string name="facebook_client_token">YOUR_CLIENT_TOKEN</string>
</resources>
```
> Find `CLIENT_TOKEN` in Meta for Developers Console: **Settings → Advanced → Client Token**

#### `android/app/src/main/AndroidManifest.xml`
```xml
<!-- Before <application> tag -->
<uses-permission android:name="android.permission.INTERNET"/>

<!-- Inside <application> tag -->
<meta-data android:name="com.facebook.sdk.ApplicationId"
           android:value="@string/facebook_app_id"/>
<meta-data android:name="com.facebook.sdk.ClientToken"
           android:value="@string/facebook_client_token"/>
```

#### Android Key Hash (Meta Console)
- **Debug**: `keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore | openssl sha1 -binary | openssl base64`
- **Google Play App Signing**: Get SHA-1 from Google Play Console → convert to Base64 (do NOT use local keystore hash)
- Register all variants: debug, release, CI/CD

---

### 3. iOS Configuration

#### Deployment Target & Podfile
```ruby
# ios/Podfile
platform :ios, '12.0'
```

#### `ios/Runner/Info.plist`
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>fb{your-app-id}</string>
    </array>
  </dict>
</array>
<key>FacebookAppID</key>
<string>{your-app-id}</string>
<key>FacebookClientToken</key>
<string>CLIENT-TOKEN</string>
<key>FacebookDisplayName</key>
<string>{your-app-name}</string>
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>fbapi</string>
  <string>fb-messenger-share-api</string>
</array>
<!-- Required for iOS 17+ App Tracking Transparency -->
<key>NSUserTrackingUsageDescription</key>
<string>This allows us to deliver a personalized experience.</string>
```

#### ⚠️ Multi-Provider Merging (Google + Facebook)
If your app also uses Google Sign-In, **merge** `CFBundleURLSchemes` — do NOT duplicate the key:
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>fb{your-app-id}</string>
      <string>com.googleusercontent.apps.{your-google-client-id}</string>
    </array>
  </dict>
</array>
```

---

### 4. iOS 17: App Tracking Transparency (Critical)
Since iOS 17, Facebook login enters **Limited Login mode** if the user denies App Tracking Transparency (ATT).

```dart
import 'package:permission_handler/permission_handler.dart';
import 'package:flutter_facebook_auth/flutter_facebook_auth.dart';

Future<LoginResult> loginWithFacebook() async {
  // Request ATT permission first (iOS 17+ required)
  await Permission.appTrackingTransparency.request();

  // Proceed with login — enters Limited Login mode if ATT denied
  return FacebookAuth.instance.login(
    permissions: ['email', 'public_profile'],
  );
}
```

#### Limited Login Mode (v7 Breaking Change)
When ATT is denied, the SDK returns a `LimitedToken` (not a `ClassicToken`). Handle both:

```dart
final LoginResult result = await FacebookAuth.instance.login();

if (result.status == LoginStatus.success) {
  final token = result.accessToken!;
  if (token is ClassicToken) {
    // Full access: use token.tokenString to call Graph API
  } else if (token is LimitedToken) {
    // Limited access: only basic profile info available
    // Cannot make Graph API calls with this token
  }
}
```

---

### 5. Implementation Flow
```dart
import 'package:flutter_facebook_auth/flutter_facebook_auth.dart';

class FacebookAuthRepository {
  // Check if already logged in
  Future<bool> isLoggedIn() async {
    final token = await FacebookAuth.instance.accessToken;
    return token != null;
  }

  // Login
  Future<AccessToken?> login() async {
    final result = await FacebookAuth.instance.login(
      permissions: ['email', 'public_profile'],
    );
    if (result.status == LoginStatus.success) return result.accessToken;
    return null;
  }

  // Get user data after login
  Future<Map<String, dynamic>> getUserData() async {
    return FacebookAuth.instance.getUserData(
      fields: 'email,name,picture.width(200)',
    );
  }

  // Logout
  Future<void> logOut() => FacebookAuth.instance.logOut();
}
```

#### Web Initialization
For Web, initialize with the App ID before any login calls:
```dart
await FacebookAuth.instance.webAndDesktopInitialize(
  appId: "YOUR_APP_ID",
  cookie: true,
  xfbml: true,
  version: "v15.0",
);
```

---

## 🛠️ Troubleshooting (FAQ)

### Q: `No implementation found` error on Android?
**A:** `strings.xml` or `AndroidManifest.xml` is not configured. The Facebook SDK crashes on startup and locks all other plugins. Fix: add both `facebook_app_id` AND `facebook_client_token` to `strings.xml`, and both `<meta-data>` tags to `AndroidManifest.xml`.

### Q: Login works in debug but fails in production?
**A:** Production builds use a different keystore. If using Google Play App Signing, get the SHA-1 from Play Console (not your local keystore) and register it in Meta Console as a Key Hash.

### Q: User's email is null even with `email` permission?
**A:** Users may choose not to share their email during the Facebook login dialog. The `email` field will be `null`. Your app must handle this gracefully (e.g., prompt user to enter email manually).

---

---

## 🎯 Minimal Viable Example

### Complete Working Implementation
```dart
import 'package:flutter/material.dart';
import 'package:flutter_facebook_auth/flutter_facebook_auth.dart';
import 'package:permission_handler/permission_handler.dart';

class FacebookAuthService {
  // Check if user is logged in
  Future<bool> isLoggedIn() async {
    final token = await FacebookAuth.instance.accessToken;
    return token != null;
  }

  // Sign-in with iOS 17 ATT handling
  Future<FacebookSignInResult> signIn() async {
    try {
      // Request App Tracking Transparency (iOS 17+)
      await Permission.appTrackingTransparency.request();

      // Perform login
      final LoginResult result = await FacebookAuth.instance.login(
        permissions: ['email', 'public_profile'],
      );

      return _handleLoginResult(result);
      
    } on Exception catch (e) {
      return FacebookSignInResult.error('Sign-in failed: $e');
    }
  }

  // Handle different login result statuses
  FacebookSignInResult _handleLoginResult(LoginResult result) {
    switch (result.status) {
      case LoginStatus.success:
        final token = result.accessToken!;
        
        // Check token type (iOS 17 Limited Login)
        if (token is ClassicToken) {
          return FacebookSignInResult.success(
            token: token,
            isLimited: false,
          );
        } else if (token is LimitedToken) {
          return FacebookSignInResult.success(
            token: token,
            isLimited: true,
          );
        }
        return FacebookSignInResult.error('Unknown token type');

      case LoginStatus.cancelled:
        return FacebookSignInResult.canceled();

      case LoginStatus.failed:
        return FacebookSignInResult.error(
          result.message ?? 'Login failed',
        );

      case LoginStatus.operationInProgress:
        return FacebookSignInResult.error('Operation already in progress');

      default:
        return FacebookSignInResult.error('Unknown status');
    }
  }

  // Get user data after successful login
  Future<Map<String, dynamic>?> getUserData() async {
    try {
      return await FacebookAuth.instance.getUserData(
        fields: 'email,name,picture.width(200),first_name,last_name',
      );
    } catch (e) {
      print('Failed to get user data: $e');
      return null;
    }
  }

  // Logout
  Future<void> logout() async {
    await FacebookAuth.instance.logOut();
  }

  // Get current access token
  Future<AccessToken?> getCurrentToken() async {
    return FacebookAuth.instance.accessToken;
  }
}

// Result class for type-safe handling
class FacebookSignInResult {
  final AccessToken? token;
  final bool isLimited;
  final String? error;
  final bool isCanceled;

  FacebookSignInResult._({
    this.token,
    this.isLimited = false,
    this.error,
    this.isCanceled = false,
  });

  factory FacebookSignInResult.success({
    required AccessToken token,
    required bool isLimited,
  }) {
    return FacebookSignInResult._(token: token, isLimited: isLimited);
  }

  factory FacebookSignInResult.canceled() {
    return FacebookSignInResult._(isCanceled: true);
  }

  factory FacebookSignInResult.error(String message) {
    return FacebookSignInResult._(error: message);
  }

  bool get isSuccess => token != null;
}

// UI Widget Example
class FacebookSignInButton extends StatefulWidget {
  final FacebookAuthService authService;

  const FacebookSignInButton({required this.authService});

  @override
  _FacebookSignInButtonState createState() => _FacebookSignInButtonState();
}

class _FacebookSignInButtonState extends State<FacebookSignInButton> {
  bool _isLoading = false;

  @override
  Widget build(BuildContext context) {
    return ElevatedButton.icon(
      onPressed: _isLoading ? null : _handleSignIn,
      icon: _isLoading
          ? SizedBox(
              width: 16,
              height: 16,
              child: CircularProgressIndicator(strokeWidth: 2),
            )
          : Icon(Icons.facebook),
      label: Text('Sign in with Facebook'),
      style: ElevatedButton.styleFrom(
        backgroundColor: Color(0xFF1877F2), // Facebook blue
      ),
    );
  }

  Future<void> _handleSignIn() async {
    setState(() => _isLoading = true);

    final result = await widget.authService.signIn();

    setState(() => _isLoading = false);

    if (!mounted) return;

    if (result.isSuccess) {
      if (result.isLimited) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('Signed in with Limited Login mode'),
            backgroundColor: Colors.orange,
          ),
        );
      } else {
        final userData = await widget.authService.getUserData();
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('Welcome ${userData?['name'] ?? 'User'}!'),
          ),
        );
      }
    } else if (!result.isCanceled && result.error != null) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('Error: ${result.error}'),
          backgroundColor: Colors.red,
        ),
      );
    }
  }
}

// Web initialization (call before any login)
Future<void> initializeFacebookWeb() async {
  await FacebookAuth.instance.webAndDesktopInitialize(
    appId: "YOUR_APP_ID",
    cookie: true,
    xfbml: true,
    version: "v15.0",
  );
}
```

---

## 🚨 Error Handling Patterns

### Common Error Scenarios & Solutions

| Error | Description | Solution |
|-------|-------------|----------|
| `No implementation found` | Missing config in `strings.xml` or `AndroidManifest.xml` | Add `facebook_app_id` AND `facebook_client_token` |
| `LoginStatus.failed` | Generic login failure | Check network, app configuration, key hash |
| `LoginStatus.cancelled` | User closed dialog | Handle gracefully, no retry |
| `Invalid key hash` | Production keystore mismatch | Use Play Console SHA-1 for Play App Signing |
| `Email is null` | User didn't share email | Handle null email gracefully |
| Limited Login mode | ATT denied on iOS 17+ | Adapt features for limited access |

### Comprehensive Error Handler
```dart
class FacebookErrorHandler {
  static String getUserMessage(LoginResult result) {
    switch (result.status) {
      case LoginStatus.cancelled:
        return 'Sign-in was canceled';
      case LoginStatus.failed:
        final message = result.message ?? '';
        
        if (message.contains('network')) {
          return 'Network error. Check your connection.';
        }
        if (message.contains('key hash')) {
          return 'Configuration error. Please update the app.';
        }
        return 'Sign-in failed. Please try again.';
        
      case LoginStatus.operationInProgress:
        return 'Sign-in already in progress';
      default:
        return 'An error occurred';
    }
  }

  static bool shouldRetry(LoginStatus status) {
    return status == LoginStatus.failed;
  }

  static bool isConfigurationError(String? message) {
    if (message == null) return false;
    return message.contains('key hash') ||
           message.contains('App ID') ||
           message.contains('configuration');
  }
}
```

### Limited Login Mode Handler (iOS 17+)
```dart
class LimitedLoginHandler {
  // Check if token is limited
  bool isLimitedToken(AccessToken token) {
    return token is LimitedToken;
  }

  // Adapt features based on token type
  Future<void> handleTokenType(AccessToken token) async {
    if (token is ClassicToken) {
      // Full access: can use Graph API
      await _fetchExtendedProfile(token.tokenString);
      await _fetchFriendsList(token.tokenString);
    } else if (token is LimitedToken) {
      // Limited access: basic profile only
      // Cannot make Graph API calls
      // Can only use basic user info from getUserData()
      await _handleLimitedAccess();
    }
  }

  Future<void> _handleLimitedAccess() async {
    // Show UI indicating limited features
    // Suggest enabling tracking for full experience
  }

  Future<void> _fetchExtendedProfile(String token) async {
    // Use token with Graph API
  }

  Future<void> _fetchFriendsList(String token) async {
    // Use token with Graph API
  }
}
```

---

## ⚠️ Common Gotchas

### 1. **`No implementation found` (Android Startup Crash)**
**Cause**: Missing `strings.xml` or incomplete `AndroidManifest.xml` configuration.

**Solution**:
```xml
<!-- android/app/src/main/res/values/strings.xml -->
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="facebook_app_id">123456789012345</string>
    <string name="facebook_client_token">abc123def456</string>
</resources>

<!-- android/app/src/main/AndroidManifest.xml -->
<application>
    <meta-data 
        android:name="com.facebook.sdk.ApplicationId"
        android:value="@string/facebook_app_id"/>
    <meta-data 
        android:name="com.facebook.sdk.ClientToken"
        android:value="@string/facebook_client_token"/>
</application>
```

**Critical**: Both `app_id` AND `client_token` are required. Missing either causes immediate crash.

### 2. **Production Login Fails (Key Hash Mismatch)**
**Cause**: Using debug key hash for production build.

**Solution**:
```bash
# For Google Play App Signing (MOST COMMON):
# 1. Go to Play Console → Setup → App signing
# 2. Copy "App signing key certificate" SHA-1
# 3. Convert to Base64:
echo "AA:BB:CC:DD:..." | xxd -r -p | openssl base64

# For local keystore (NOT recommended for production):
keytool -exportcert -alias YOUR_ALIAS -keystore release.keystore | \
  openssl sha1 -binary | openssl base64
```

Register ALL variants in Meta Console:
- Debug key hash (for development)
- Release key hash (Play Console SHA-1)
- CI/CD key hash (if applicable)

### 3. **Email is Null Even with Permission**
**Cause**: User chose not to share email during login.

**Solution**:
```dart
final userData = await FacebookAuth.instance.getUserData(
  fields: 'email,name,picture',
);

if (userData['email'] == null) {
  // User declined to share email
  // Option 1: Request manually
  final email = await showEmailDialog();
  
  // Option 2: Use Facebook ID as identifier
  await createAccount(
    fbId: userData['id'],
    name: userData['name'],
  );
}
```

### 4. **iOS 17 Limited Login Mode**
**Cause**: User denied App Tracking Transparency.

**Solution**:
```dart
// Always request ATT before login
final attStatus = await Permission.appTrackingTransparency.request();

if (attStatus.isDenied) {
  // Warn user about limited features
  showDialog(
    context: context,
    builder: (_) => AlertDialog(
      title: Text('Limited Login'),
      content: Text(
        'Without tracking permission, some features will be limited.'
      ),
    ),
  );
}

// Proceed with login (will use Limited Login mode)
final result = await FacebookAuth.instance.login(...);
```

### 5. **Multi-Provider URL Scheme Conflict**
**Cause**: Duplicate `CFBundleURLTypes` in `Info.plist`.

**Solution**:
```xml
<!-- ❌ WRONG: Duplicate keys -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>fb123456</string></array>
  </dict>
</array>
<key>CFBundleURLTypes</key>  <!-- DUPLICATE! -->
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>com.googleusercontent...</string></array>
  </dict>
</array>

<!-- ✅ CORRECT: Merge into one array -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>fb123456</string>
      <string>com.googleusercontent.apps.123-abc</string>
    </array>
  </dict>
</array>
```

### 6. **Privacy Policy Required**
**Cause**: Facebook requires privacy policy URL even for testing.

**Solution**:
- Add Privacy Policy URL in Meta Console → Settings → Basic
- Use placeholder during development: `https://example.com/privacy`
- Must be a real, accessible URL for production

### 7. **Web Login Fails Silently**
**Cause**: Forgot to initialize Facebook SDK for web.

**Solution**:
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize for web before runApp
  if (kIsWeb) {
    await FacebookAuth.instance.webAndDesktopInitialize(
      appId: "YOUR_APP_ID",
      cookie: true,
      xfbml: true,
      version: "v15.0",
    );
  }
  
  runApp(MyApp());
}
```

### 8. **Token Expiration Not Handled**
**Cause**: Access tokens expire after ~60 days.

**Solution**:
```dart
class TokenRefreshService {
  Future<bool> refreshTokenIfNeeded() async {
    final token = await FacebookAuth.instance.accessToken;
    
    if (token == null) return false;
    
    // Check if token is near expiration (e.g., within 7 days)
    final expiresAt = token.expiresAt;
    if (expiresAt != null) {
      final daysUntilExpiry = 
          expiresAt.difference(DateTime.now()).inDays;
      
      if (daysUntilExpiry < 7) {
        // Re-login to get fresh token
        await FacebookAuth.instance.logOut();
        // Prompt user to log in again
        return false;
      }
    }
    
    return true;
  }
}
```

### 9. **Graph API Calls Fail with Limited Token**
**Cause**: Limited tokens cannot access Graph API.

**Solution**:
```dart
Future<Map<String, dynamic>?> fetchUserData(AccessToken token) async {
  if (token is LimitedToken) {
    // Limited token: use getUserData() instead of Graph API
    return await FacebookAuth.instance.getUserData();
  }
  
  if (token is ClassicToken) {
    // Classic token: can use Graph API
    final response = await http.get(
      Uri.parse(
        'https://graph.facebook.com/me?fields=email,name,picture'
        '&access_token=${token.tokenString}'
      ),
    );
    
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
  }
  
  return null;
}
```

---

## Constraints
* **Client Token Required**: v7 requires `facebook_client_token` in both `strings.xml` (Android) and `Info.plist` (iOS). Missing it causes SDK initialization failure.
* **Privacy Policy URL**: Even for development testing, a valid Privacy Policy URL is required to set your Facebook App to "Live" mode.
* **iOS Swift Requirement**: This plugin cannot be used in Objective-C Flutter projects.
* **ATT before Login (iOS 17+)**: Always request `appTrackingTransparency` permission before triggering Facebook login to avoid Limited Login mode where possible.
