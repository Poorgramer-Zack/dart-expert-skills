---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Supabase Initialization and Deep Linking

## Core Global Setup
Supabase requires global initialization, typically housed cleanly within `main.dart`. Unlike Firebase, Supabase provides its own global getter via `Supabase.instance.client`, negating the need for extensive dependency injection of the client itself.

```dart
import 'package:supabase_flutter/supabase_flutter.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Supabase.initialize(
    url: 'YOUR_SUPABASE_URL', // Get from Project Settings -> API
    anonKey: 'YOUR_SUPABASE_ANON_KEY', // Public privileges only; safe to embed!
  );

  runApp(const MyApp());
}

// Global accessor best practice
final supabase = Supabase.instance.client;
```

## Crucial Requirement: Deep Links (`app_links`)
Supabase heavily relies on returning to the application via Deep Links for critical authentication flows:
1. **Magic Link Login (OTP)**
2. **Confirm Email (Sign Up)**
3. **Resetting Passwords**
4. **Calling `.signInWithOAuth()`**

The `supabase_flutter` SDK uses the `app_links` package internally. It is absolutely mandatory to configure Deep Links on the iOS/Android side and add the respective Reverse URI to the Supabase Dashboard.

### Dashboard Configuration
1. Go to Authentication -> URL Configuration in Supabase.
2. In the "Redirect URLs" section, add a custom scheme, e.g., `io.myapp.flutter://login-callback`.
   * *Note*: This scheme serves merely as a gateway; the Flutter package intercepts it inherently.

### Flutter App Implementation
Configure your deep link handlers via Native files targeting the same scheme. While Supabase handles the session extraction automatically upon link opened, the native OS must still physically route the URL intent toward the Flutter Engine correctly via `AndroidManifest.xml` (using `<data android:scheme="io.myapp.flutter" android:host="login-callback" />`) and `Info.plist` (using `CFBundleURLSchemes`).
