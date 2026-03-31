---
name: "shared-preferences-best-practices"
description: "Best practices for implementing key-value persistence in Flutter using shared_preferences. Includes static wrapper patterns and Riverpod 3.0 reactive integration."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Shared Preferences Best Practices

## Goal
Provide a simple, persistent storage solution for primitive data types. By encapsulating SharedPreferences into a static wrapper and integrating it with Riverpod 3.0, we ensure easy access throughout the app while maintaining UI reactivity.

## Instructions

### 1. Installation
```yaml
dependencies:
  shared_preferences: ^2.3.5
  flutter_riverpod: ^3.0.0-beta.0 # Or latest 3.x
  riverpod_annotation: ^4.0.0
```

### 2. Static Wrapper Pattern (Encapsulation)
Encapsulating SharedPreferences into a static class allows for clean, synchronous reads and easy maintenance of key constants.

```dart
import 'package:shared_preferences/shared_preferences.dart';

class Prefs {
  static late SharedPreferences _prefs;

  // Key constants
  static const _keyIsDarkMode = 'is_dark_mode';
  static const _keyUserName = 'user_name';

  /// Initialize the instance. Call this in main() before runApp().
  static Future<void> init() async {
    _prefs = await SharedPreferences.getInstance();
  }

  // Getters & Setters
  static bool get isDarkMode => _prefs.getBool(_keyIsDarkMode) ?? false;
  static Future<bool> setDarkMode(bool value) => _prefs.setBool(_keyIsDarkMode, value);

  static String get userName => _prefs.getString(_keyUserName) ?? '';
  static Future<bool> setUserName(String value) => _prefs.setString(_keyUserName, value);
  
  /// Clear all data
  static Future<bool> clear() => _prefs.clear();
}
```

### 3. Application Startup
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize static preferences before app starts
  await Prefs.init();
  
  runApp(const ProviderScope(child: MyApp()));
}
```

### 4. Riverpod 3.0 Reactive Integration
Since SharedPreferences does not natively notify listeners, we wrap the static calls in a Riverpod `Notifier` to provide reactive state to the UI.

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'prefs.dart';

part 'settings_provider.g.dart';

@riverpod
class DarkModeState extends _$DarkModeState {
  @override
  bool build() {
    // Read initial value from static wrapper
    return Prefs.isDarkMode;
  }

  Future<void> toggle() async {
    final newValue = !state;
    // 1. Update persistent storage
    await Prefs.setDarkMode(newValue);
    // 2. Update reactive state (UI will rebuild)
    state = newValue;
  }
}
```

## Constraints
* **Static Initialization**: `Prefs.init()` MUST be awaited in `main()` before `runApp`.
* **Encapsulation**: Prohibit direct calls to `SharedPreferences.getInstance()` outside the `Prefs` class to prevent key string fragmentation.
* **Reactivity**: Use Riverpod `Notifier`s to wrap `Prefs` values when the UI needs to automatically rebuild upon change.
* **Primitive Only**: Only store `String`, `int`, `bool`, `double`, and `List<String>`.
