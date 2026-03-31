---
name: "flutter-isolate"
description: "Dart Isolate patterns for background processing using Isolate.run() and compute(). Use when offloading heavy tasks (JSON parsing, image processing, encryption) from the main thread to prevent UI jank."
metadata:
  last_modified: "2026-03-13 14:16:00 (GMT+8)"
---

# Flutter Isolate

## Goal
Move heavy computations off the main isolate to prevent UI jank (>16ms frame gaps), using the simplest API that fits the use case.

## Decision: Which API to Use?

| Scenario | API |
|---|---|
| One-shot, single return value | `Isolate.run()` |
| Web + mobile cross-platform | `compute()` |
| Repeated calls or multiple responses | `Isolate.spawn()` + `ReceivePort`/`SendPort` |
| Need platform plugins in background | `BackgroundIsolateBinaryMessenger` |

---

## 1. Short-lived Isolate: `Isolate.run()`

Best for single, one-off computations. The isolate spawns, runs the task, returns the value, then shuts down automatically.

```dart
// Decode a large JSON file without blocking the UI thread
Future<List<Photo>> getPhotos() async {
  // Load asset on the main isolate first (rootBundle not accessible in isolate)
  final String jsonString = await rootBundle.loadString('assets/photos.json');

  // Offload CPU-heavy decoding to a new isolate
  final List<Photo> photos = await Isolate.run<List<Photo>>(() {
    final List<Object?> photoData = jsonDecode(jsonString) as List<Object?>;
    return photoData.cast<Map<String, Object?>>().map(Photo.fromJson).toList();
  });

  return photos;
}
```

> **Key behavior**: The result is *transferred* (not copied) back to the main isolate via `Isolate.exit` internally — zero-copy for the return value.

---

## 2. Cross-platform: `compute()`

`compute()` is Flutter's wrapper that falls back gracefully on **Flutter Web** (runs on the main thread there, since web doesn't support isolates).

```dart
// Equivalent to Isolate.run on mobile/desktop, runs on main thread on web
Future<List<Photo>> getPhotos(String jsonString) async {
  return compute(_parsePhotos, jsonString);
}

// Top-level or static function only — closures are NOT supported by compute()
List<Photo> _parsePhotos(String jsonString) {
  final data = jsonDecode(jsonString) as List<Object?>;
  return data.cast<Map<String, Object?>>().map(Photo.fromJson).toList();
}
```

> **Constraint**: The callback must be a **top-level or static** function. Closures are not supported.

---

## 3. Long-lived Background Worker

Use `Isolate.spawn()` + ports when you need to send multiple requests to the same isolate over time (avoids spawn overhead per call). See `references/long_lived_worker.md` for the full robust implementation with error handling, message sequencing, and port cleanup.

Quick structure overview:

```dart
class Worker {
  final SendPort _commands;     // main → worker
  final ReceivePort _responses; // worker → main
  final Map<int, Completer<Object?>> _activeRequests = {};
  int _idCounter = 0;
  bool _closed = false;

  static Future<Worker> spawn() async { /* ... */ }
  Future<Object?> parseJson(String message) async { /* ... */ }
  void close() { _closed = true; _responses.close(); }
}
```

The two-way port handshake pattern:
1. Main creates `ReceivePort`, passes its `sendPort` to `Isolate.spawn()`
2. Worker creates its own `ReceivePort`, sends its `sendPort` back
3. Both sides now have a channel — main tracks requests with `Completer` + incrementing IDs

---

## 4. Platform Plugins in Background Isolates (Flutter 3.7+)

Since Flutter 3.7, you can call platform plugins (e.g., `shared_preferences`, native crypto APIs) from background isolates using `BackgroundIsolateBinaryMessenger`.

```dart
import 'dart:isolate';
import 'package:flutter/services.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  // Must capture token on the main isolate before spawning
  final RootIsolateToken token = RootIsolateToken.instance!;
  Isolate.spawn(_isolateMain, token);
}

Future<void> _isolateMain(RootIsolateToken token) async {
  // Register BEFORE using any platform plugins
  BackgroundIsolateBinaryMessenger.ensureInitialized(token);

  final prefs = await SharedPreferences.getInstance();
  print(prefs.getBool('isDebug'));
}
```

---

## Message Passing Rules

- **Mutable objects are copied** when sent via `SendPort.send()` — mutating them in the worker does not affect the main isolate.
- **Immutable objects** (e.g., `String`, unmodifiable `Uint8List`) send a *reference* for performance.
- `Isolate.exit()` transfers ownership (zero-copy) — used internally by `Isolate.run()` and `compute()`.

---

## Limitations

| Limitation | Detail |
|---|---|
| **Web** | Isolates not supported on Flutter Web; use `compute()` as a cross-platform shim |
| **rootBundle / dart:ui** | Not accessible inside background isolates; load assets on main isolate first |
| **UI operations** | No widget or rendering calls allowed in background isolates |
| **Plugin push messages** | Cannot receive *unsolicited* messages from host platform (e.g., no Firestore listener in background isolate); you can *query* but not *subscribe* |
| **Shared mutable state** | Global variables are *copied* at spawn time — changes in the worker never reflect back |

---

## References

- `references/long_lived_worker.md` — Full robust Worker class implementation with error handling, Completer-based sequencing, and port lifecycle management.
