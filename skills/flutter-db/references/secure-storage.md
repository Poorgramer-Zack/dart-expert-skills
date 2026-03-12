---
name: "flutter-secure-storage-best-practices"
description: "Best practices for implementing encrypted key-value persistence in Flutter using flutter_secure_storage. Use this for tokens, passwords, and sensitive keys."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Secure Storage Best Practices

## Goal
Provide an encrypted storage solution for sensitive data like OAuth tokens, private keys, and session passwords. Unlike SharedPreferences, Secure Storage uses platform-specific encrypted stores (Keychain for iOS, Keystore for Android) to protect data at rest.

## Instructions

### 1. Installation
```yaml
dependencies:
  flutter_secure_storage: ^10.0.0
```

### 2. Static Wrapper Pattern (Security Encapsulation)
Centralizing secure storage access ensures consistent use of encryption options and prevents key collisions.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecurePrefs {
  static late FlutterSecureStorage _storage;

  // Key constants
  static const _keyAccessToken = 'access_token';
  static const _keySecretKey = 'app_secret_key';

  static void init() {
    _storage = const FlutterSecureStorage(
      // Android: use EncryptedSharedPreferences for better security
      aOptions: AndroidOptions(
        encryptedSharedPreferences: true,
      ),
      // iOS: use keychain search and accessibility constants if needed
      iOptions: IOSOptions(
        accessibility: KeychainAccessibility.first_unlock,
      ),
    );
  }

  // Safe reading (returns null if not found)
  static Future<String?> get accessToken => _storage.read(key: _keyAccessToken);
  static Future<void> setAccessToken(String value) => _storage.write(key: _keyAccessToken, value: value);

  static Future<String?> get secretKey => _storage.read(key: _keySecretKey);
  static Future<void> setSecretKey(String value) => _storage.write(key: _keySecretKey, value: value);

  static Future<void> deleteAccessToken() => _storage.delete(key: _keyAccessToken);
  static Future<void> clearAll() => _storage.deleteAll();
}
```

### 3. Application Startup
Initializing Secure Storage is synchronous as the instance itself is a wrapper, but `read/write` operations are always asynchronous.

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  SecurePrefs.init(); // Initialize the wrapper
  runApp(const ProviderScope(child: MyApp()));
}
```

### 4. Riverpod 3.0 Integration (Async Persistence)
Since secure storage involves asynchronous I/O, use `@riverpod` with `Future` for initial loading or handle the state via a dedicated `AuthNotifier`.

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'secure_prefs.dart';

part 'auth_state_provider.g.dart';

@riverpod
class AuthToken extends _$AuthToken {
  @override
  Future<String?> build() async {
    // Load initial token asynchronously from secure storage
    return await SecurePrefs.accessToken;
  }

  Future<void> updateToken(String token) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      await SecurePrefs.setAccessToken(token);
      return token;
    });
  }

  Future<void> logout() async {
    await SecurePrefs.deleteAccessToken();
    state = const AsyncValue.data(null);
  }
}
```

## Constraints
* **Encryption Always**: On Android, always set `encryptedSharedPreferences: true` for the best security standards.
* **Sensitive Data ONLY**: Do not store non-sensitive settings (like theme mode) in Secure Storage; it is significantly slower than SharedPreferences due to encryption overhead.
* **Asynchronous Integrity**: All `read`, `write`, and `delete` operations are `Future` based. Never ignore the results of these operations in critical flows.
* **Keychain Strategy**: On iOS, be aware of `KeychainAccessibility` levels; `first_unlock` is usually the best balance for background operations.
