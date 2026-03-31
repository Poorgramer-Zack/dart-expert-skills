---
name: "flutter-line-login"
description: "Best practices for implementing LINE Login in Flutter using flutter_line_sdk. Includes Android/iOS configuration, channel setup, session management, minimal viable examples, comprehensive error handling patterns, and common gotchas."
metadata:
  last_modified: "2026-03-31 20:19:44 (GMT+8)"
---

# LINE Login Best Practices

## Goal
Establish a reliable LINE Login implementation for Flutter applications. This guide covers integration with the LINE Developers Console, platform-specific native configurations, and session lifecycle management using the `flutter_line_sdk`.

## Process

### Installation
```yaml
dependencies:
  flutter_line_sdk: ^7.1.2
```

### Provider Configuration
#### iOS (`Info.plist` & Deployment Target)
1.  **Deployment Target**: Must be iOS **13.0** or higher.
    - Update `ios/Podfile`: `platform :ios, '13.0'`
2.  **Info.plist**: Add `CFBundleURLTypes` and `LSApplicationQueriesSchemes`.
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <!-- line3rdp.$(PRODUCT_BUNDLE_IDENTIFIER) -->
      <string>line3rdp.com.your.bundle.id</string>
    </array>
  </dict>
</array>
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>lineauth2</string>
</array>
```

#### Android (`build.gradle`)
1.  **Min SDK**: Must be **24** or higher.
```gradle
android {
    defaultConfig {
        minSdk 24
    }
}
```
2.  **Package Name**: Ensure the Android package name matches the one registered in LINE Developers Console.

### Implementation Flow
#### Initialization
Call `setup` exactly once before any other SDK methods.

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await LineSDK.instance.setup("YOUR_CHANNEL_ID");
  runApp(const MyApp());
}
```

#### Login Flow
```dart
import 'package:flutter_line_sdk/flutter_line_sdk.dart';

class LineAuthRepository {
  Future<LoginResult?> login() async {
    try {
      // Default scope is ["profile"]
      final result = await LineSDK.instance.login(
        scopes: ["profile", "openid", "email"],
      );
      return result;
    } on PlatformException catch (e) {
      // Handle login error
      print(e.message);
      return null;
    }
  }

  Future<void> logout() async {
    try {
      await LineSDK.instance.logout();
    } catch (e) {
      print(e);
    }
  }
}
```

---

## 🛠️ Troubleshooting (FAQ)

### Q: Why do I get `No implementation found` on Android?
**A:** This often happens if the `minSdk` is below 24 or if the LINE SDK fails to initialize due to incorrect package name/signature configuration in the LINE Developers Console.

### Q: How do I get the user's email?
**A:** You must request the `email` scope during login. Note that users can choose not to share their email, so `result.accessToken.email` might be null even if the scope was requested.

---

---

## 🎯 Minimal Viable Example

### Complete Working Implementation
```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_line_sdk/flutter_line_sdk.dart';

class LineAuthService {
  static const String _channelId = 'YOUR_CHANNEL_ID';
  
  // Initialize LINE SDK (call once in main.dart)
  static Future<void> initialize() async {
    try {
      await LineSDK.instance.setup(_channelId);
      print('✅ LINE SDK initialized');
    } on PlatformException catch (e) {
      print('❌ LINE SDK initialization failed: ${e.message}');
      rethrow;
    }
  }

  // Sign-in with comprehensive error handling
  Future<LineSignInResult> signIn({
    List<String> scopes = const ['profile', 'openid', 'email'],
  }) async {
    try {
      final result = await LineSDK.instance.login(scopes: scopes);
      
      // Save user info immediately
      await _saveUserInfo(result);
      
      return LineSignInResult.success(result);
      
    } on PlatformException catch (e) {
      return _handlePlatformException(e);
    } catch (e) {
      return LineSignInResult.error('Unexpected error: $e');
    }
  }

  // Get current access token
  Future<StoredAccessToken?> getCurrentAccessToken() async {
    try {
      return await LineSDK.instance.currentAccessToken;
    } catch (e) {
      return null;
    }
  }

  // Check if user is logged in
  Future<bool> isLoggedIn() async {
    final token = await getCurrentAccessToken();
    return token != null;
  }

  // Get user profile
  Future<UserProfile?> getUserProfile() async {
    try {
      return await LineSDK.instance.getProfile();
    } on PlatformException catch (e) {
      print('Failed to get profile: ${e.message}');
      return null;
    }
  }

  // Logout
  Future<void> logout() async {
    try {
      await LineSDK.instance.logout();
      print('👋 Logged out successfully');
    } on PlatformException catch (e) {
      print('Logout failed: ${e.message}');
    }
  }

  // Verify access token validity
  Future<bool> verifyAccessToken() async {
    try {
      final result = await LineSDK.instance.verifyAccessToken();
      return result.data != null;
    } catch (e) {
      return false;
    }
  }

  // Refresh access token
  Future<StoredAccessToken?> refreshAccessToken() async {
    try {
      final result = await LineSDK.instance.refreshAccessToken();
      return result.value;
    } on PlatformException catch (e) {
      print('Token refresh failed: ${e.message}');
      return null;
    }
  }

  // Handle platform exceptions
  LineSignInResult _handlePlatformException(PlatformException e) {
    final code = e.code;
    final message = e.message ?? 'Unknown error';

    // Common error codes
    switch (code) {
      case 'CANCEL':
        return LineSignInResult.canceled();
      case 'AUTHENTICATION_AGENT_ERROR':
        return LineSignInResult.error('Authentication failed: $message');
      case 'NETWORK_ERROR':
        return LineSignInResult.error('Network error. Check connection.');
      case 'SERVER_ERROR':
        return LineSignInResult.error('Server error. Try again later.');
      case 'INTERNAL_ERROR':
        return LineSignInResult.error('Internal error: $message');
      default:
        return LineSignInResult.error('Error ($code): $message');
    }
  }

  // Save user information
  Future<void> _saveUserInfo(LoginResult result) async {
    // Save to secure storage or database
    final profile = result.userProfile;
    if (profile != null) {
      print('User: ${profile.displayName}');
      print('User ID: ${profile.userId}');
      // Store in your database
    }
  }
}

// Result class for type-safe handling
class LineSignInResult {
  final LoginResult? loginResult;
  final String? error;
  final bool isCanceled;

  LineSignInResult._({
    this.loginResult,
    this.error,
    this.isCanceled = false,
  });

  factory LineSignInResult.success(LoginResult result) {
    return LineSignInResult._(loginResult: result);
  }

  factory LineSignInResult.canceled() {
    return LineSignInResult._(isCanceled: true);
  }

  factory LineSignInResult.error(String message) {
    return LineSignInResult._(error: message);
  }

  bool get isSuccess => loginResult != null;
}

// UI Widget Example
class LineSignInButton extends StatefulWidget {
  final LineAuthService authService;

  const LineSignInButton({required this.authService});

  @override
  _LineSignInButtonState createState() => _LineSignInButtonState();
}

class _LineSignInButtonState extends State<LineSignInButton> {
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
          : Icon(Icons.login),
      label: Text('Sign in with LINE'),
      style: ElevatedButton.styleFrom(
        backgroundColor: Color(0xFF00B900), // LINE green
      ),
    );
  }

  Future<void> _handleSignIn() async {
    setState(() => _isLoading = true);

    final result = await widget.authService.signIn();

    setState(() => _isLoading = false);

    if (!mounted) return;

    if (result.isSuccess) {
      final profile = result.loginResult!.userProfile;
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('Welcome ${profile?.displayName ?? 'User'}!'),
        ),
      );
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

// Main app initialization
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize LINE SDK before runApp
  await LineAuthService.initialize();
  
  runApp(MyApp());
}
```

---

## 🚨 Error Handling Patterns

### Common Error Codes & Solutions

| Error Code | Description | Solution |
|------------|-------------|----------|
| `CANCEL` | User canceled sign-in | Handle gracefully, no retry |
| `AUTHENTICATION_AGENT_ERROR` | LINE app issue or config error | Check Channel ID, reinstall LINE app |
| `NETWORK_ERROR` | No internet connection | Show offline UI, enable retry |
| `SERVER_ERROR` | LINE server temporary issue | Retry after delay |
| `INTERNAL_ERROR` | SDK internal error | Check configuration, update SDK |
| `No implementation found` | minSdk < 24 or missing setup | Set minSdk to 24+, call setup() |

### Comprehensive Error Handler
```dart
class LineErrorHandler {
  static String getUserMessage(PlatformException error) {
    switch (error.code) {
      case 'CANCEL':
        return 'Sign-in was canceled';
      case 'AUTHENTICATION_AGENT_ERROR':
        return 'Authentication failed. Please try again.';
      case 'NETWORK_ERROR':
        return 'No internet connection. Check your network.';
      case 'SERVER_ERROR':
        return 'Server error. Please try again later.';
      case 'INTERNAL_ERROR':
        return 'An internal error occurred.';
      default:
        return error.message ?? 'An error occurred';
    }
  }

  static bool shouldRetry(String errorCode) {
    return errorCode == 'NETWORK_ERROR' || 
           errorCode == 'SERVER_ERROR';
  }

  static bool isConfigurationError(String errorCode) {
    return errorCode == 'AUTHENTICATION_AGENT_ERROR' ||
           errorCode == 'INTERNAL_ERROR';
  }

  // Retry logic with exponential backoff
  static Future<T?> retryOperation<T>({
    required Future<T> Function() operation,
    int maxAttempts = 3,
    Duration initialDelay = const Duration(seconds: 1),
  }) async {
    int attempt = 0;
    Duration delay = initialDelay;

    while (attempt < maxAttempts) {
      try {
        return await operation();
      } on PlatformException catch (e) {
        attempt++;
        
        if (!shouldRetry(e.code) || attempt >= maxAttempts) {
          rethrow;
        }

        print('Retry attempt $attempt after ${delay.inSeconds}s...');
        await Future.delayed(delay);
        delay *= 2; // Exponential backoff
      }
    }

    return null;
  }
}

// Usage example
Future<void> signInWithRetry() async {
  final result = await LineErrorHandler.retryOperation(
    operation: () => lineAuthService.signIn(),
    maxAttempts: 3,
  );
}
```

### Token Management
```dart
class LineTokenManager {
  // Check token validity and refresh if needed
  Future<bool> ensureValidToken(LineAuthService authService) async {
    try {
      // Check if token exists
      if (!await authService.isLoggedIn()) {
        return false;
      }

      // Verify token is valid
      if (!await authService.verifyAccessToken()) {
        // Try to refresh
        final newToken = await authService.refreshAccessToken();
        return newToken != null;
      }

      return true;
    } catch (e) {
      print('Token validation failed: $e');
      return false;
    }
  }

  // Handle expired token scenario
  Future<void> handleExpiredToken({
    required BuildContext context,
    required LineAuthService authService,
  }) async {
    final shouldRefresh = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Session Expired'),
        content: Text('Your session has expired. Please sign in again.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: Text('Cancel'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: Text('Sign In'),
          ),
        ],
      ),
    );

    if (shouldRefresh == true) {
      await authService.logout();
      await authService.signIn();
    }
  }
}
```

---

## ⚠️ Common Gotchas

### 1. **`No implementation found` on Android**
**Cause**: minSdk below 24 or LINE SDK not initialized.

**Solution**:
```gradle
// android/app/build.gradle
android {
    defaultConfig {
        minSdk 24  // MUST be 24 or higher
        targetSdk 34
    }
}
```

```dart
// main.dart - Initialize before runApp
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await LineSDK.instance.setup('YOUR_CHANNEL_ID');
  runApp(MyApp());
}
```

### 2. **Email is Null Even with Permission**
**Cause**: User chose not to share email or didn't verify email in LINE.

**Solution**:
```dart
final result = await LineSDK.instance.login(
  scopes: ['profile', 'openid', 'email'],
);

final profile = result.userProfile;

// Email may still be null!
if (profile?.email == null) {
  // Option 1: Request manually
  final email = await showEmailDialog();
  
  // Option 2: Use LINE userId as identifier
  await createAccount(
    lineId: profile!.userId,
    displayName: profile.displayName,
  );
}
```

### 3. **iOS URL Scheme Configuration**
**Cause**: Incorrect or missing `CFBundleURLSchemes` in Info.plist.

**Solution**:
```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <!-- Format: line3rdp.YOUR_BUNDLE_ID -->
      <string>line3rdp.com.example.myapp</string>
    </array>
  </dict>
</array>

<!-- ALSO required: -->
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>lineauth2</string>
</array>
```

**Critical**: `line3rdp.` prefix is required!

### 4. **Channel ID Mismatch**
**Cause**: Wrong Channel ID or using Messaging API channel instead of LINE Login channel.

**Solution**:
- Go to [LINE Developers Console](https://developers.line.biz/console/)
- Create a **LINE Login** channel (NOT Messaging API)
- Copy the **Channel ID** (not Channel Secret)
- Verify in `main.dart`:

```dart
// ❌ WRONG: Using Channel Secret
await LineSDK.instance.setup('abc123secretkey');

// ✅ CORRECT: Using Channel ID
await LineSDK.instance.setup('1234567890');
```

### 5. **Android Package Name Mismatch**
**Cause**: Package name in app doesn't match LINE Console configuration.

**Solution**:
```kotlin
// android/app/build.gradle
android {
    defaultConfig {
        applicationId "com.example.myapp"  // Must match LINE Console
    }
}
```

In LINE Developers Console:
- Settings → App settings → Android package name
- Must **exactly** match `applicationId` in build.gradle

### 6. **Multiple setup() Calls Crash App**
**Cause**: Calling `LineSDK.instance.setup()` more than once.

**Solution**:
```dart
// ❌ WRONG: Multiple calls
void main() async {
  await LineSDK.instance.setup('CHANNEL_ID');
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    LineSDK.instance.setup('CHANNEL_ID');  // CRASH!
    return MaterialApp(...);
  }
}

// ✅ CORRECT: Call only once in main()
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await LineSDK.instance.setup('CHANNEL_ID');
  runApp(MyApp());
}
```

### 7. **LINE App Not Installed (Android/iOS)**
**Cause**: LINE SDK prefers LINE app for authentication.

**Solution**:
```dart
// The SDK automatically falls back to web-based login
// if LINE app is not installed. No special handling needed.

// However, web-based login may have different UX:
try {
  final result = await LineSDK.instance.login();
  // Works regardless of LINE app installation
} on PlatformException catch (e) {
  // Handle errors
}
```

### 8. **Token Expiration Not Handled**
**Cause**: Access tokens expire after a period.

**Solution**:
```dart
class ApiClient {
  Future<Response> callApi() async {
    // Always verify token before API calls
    final isValid = await LineSDK.instance.verifyAccessToken();
    
    if (!isValid.data) {
      // Token expired, try refresh
      try {
        await LineSDK.instance.refreshAccessToken();
      } catch (e) {
        // Refresh failed, need to re-login
        await LineSDK.instance.logout();
        throw Exception('Session expired. Please login again.');
      }
    }

    // Proceed with API call
    final token = await LineSDK.instance.currentAccessToken;
    return http.get(
      Uri.parse('YOUR_API'),
      headers: {'Authorization': 'Bearer ${token!.value}'},
    );
  }
}
```

### 9. **Deployment Target Too Low (iOS)**
**Cause**: iOS deployment target below 13.0.

**Solution**:
```ruby
# ios/Podfile
platform :ios, '13.0'  # Minimum required
```

```xml
<!-- ios/Runner.xcodeproj/project.pbxproj -->
<!-- Or set in Xcode: Runner → General → Deployment Target = 13.0 -->
```

### 10. **Scope Not Granted**
**Cause**: Requesting scopes the channel doesn't have permission for.

**Solution**:
```dart
// Only request scopes enabled in LINE Console
// Go to Console → Your Channel → Permissions

// Basic setup (always available):
final result = await LineSDK.instance.login(
  scopes: ['profile'],
);

// If email/openid enabled in Console:
final result = await LineSDK.instance.login(
  scopes: ['profile', 'openid', 'email'],
);

// Verify granted scopes:
final accessToken = await LineSDK.instance.currentAccessToken;
print('Granted scopes: ${accessToken?.scopes}');
```

---

## Constraints
* **Channel ID**: Ensure the Channel ID is correctly copied from the LINE Developers Console (Line Login channel type).
* **Native Setup**: LINE SDK is sensitive to `bundleId` (iOS) and `packageName/signature` (Android). Any mismatch will cause login failures.
* **One-time Initialization**: `LineSDK.instance.setup()` must be called only once during the app lifecycle.
