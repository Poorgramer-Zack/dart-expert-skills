---
name: "flutter-facebook-login"
description: "Best practices for implementing Facebook Login using flutter_facebook_auth v7.x. Includes Android/iOS/Web configuration, iOS 17 App Tracking Transparency handling, Limited Login mode, and multi-provider Info.plist merging."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
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

## Constraints
* **Client Token Required**: v7 requires `facebook_client_token` in both `strings.xml` (Android) and `Info.plist` (iOS). Missing it causes SDK initialization failure.
* **Privacy Policy URL**: Even for development testing, a valid Privacy Policy URL is required to set your Facebook App to "Live" mode.
* **iOS Swift Requirement**: This plugin cannot be used in Objective-C Flutter projects.
* **ATT before Login (iOS 17+)**: Always request `appTrackingTransparency` permission before triggering Facebook login to avoid Limited Login mode where possible.
