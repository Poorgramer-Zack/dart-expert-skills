---
name: "flutter-line-login"
description: "Best practices for implementing LINE Login in Flutter using flutter_line_sdk. Includes Android/iOS configuration, channel setup, and session management."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
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

## Constraints
* **Channel ID**: Ensure the Channel ID is correctly copied from the LINE Developers Console (Line Login channel type).
* **Native Setup**: LINE SDK is sensitive to `bundleId` (iOS) and `packageName/signature` (Android). Any mismatch will cause login failures.
* **One-time Initialization**: `LineSDK.instance.setup()` must be called only once during the app lifecycle.
