---
metadata:
  last_modified: "2026-03-31 14:30:00 (GMT+8)"
---

# Testing Troubleshooting & Edge Cases

## Goal
Diagnose and resolve common production testing failures including async errors, golden test platform inconsistencies, missing widget dependencies, and mock configuration issues. This guide focuses on real-world edge cases encountered in CI/CD pipelines and cross-platform development.

## Instructions

### Common Widget Test Failures

#### 1. Async Operation Errors

**⚠️ CRITICAL: "Expected: \<Future\>" Error**

**Problem:** Tests fail with messages like "Expected: \<Future\<void\>\>" or assertions execute before async operations complete.

**Root Cause:** Missing `await` on `tester.pump()` or `tester.pumpAndSettle()` after triggering state changes.

**❌ Anti-pattern:**
```dart
testWidgets('Button tap updates counter', (tester) async {
  await tester.pumpWidget(MaterialApp(home: CounterWidget()));
  
  await tester.tap(find.byType(ElevatedButton));
  // ❌ Missing pump - test checks OLD state!
  expect(find.text('1'), findsOneWidget); // FAILS
});
```

**✅ Best Practice:**
```dart
testWidgets('Button tap updates counter', (tester) async {
  await tester.pumpWidget(MaterialApp(home: CounterWidget()));
  
  await tester.tap(find.byType(ElevatedButton));
  await tester.pumpAndSettle(); // ✅ Wait for all frames to complete
  expect(find.text('1'), findsOneWidget); // PASSES
});
```

**When to use each:**
- `pump()`: Single frame advance - use for precise timing control
- `pumpAndSettle()`: Wait until no more frames scheduled - use for animations/async state updates
- `pump(Duration)`: Advance virtual time by specific duration

---

#### 2. MediaQuery/Theme Missing Errors

**⚠️ CRITICAL: "MediaQuery.of() called with a context that does not contain a MediaQuery"**

**Problem:** Widgets crash during tests with errors about missing MediaQuery, Theme, or Directionality.

**Root Cause:** Test widgets not wrapped in MaterialApp or CupertinoApp, which provide these dependencies.

**❌ Anti-pattern:**
```dart
testWidgets('Widget renders correctly', (tester) async {
  // ❌ No MaterialApp wrapper
  await tester.pumpWidget(MyCustomWidget());
  // CRASH: MediaQuery.of() called with a context that does not contain a MediaQuery
});
```

**✅ Best Practice:**
```dart
testWidgets('Widget renders correctly', (tester) async {
  // ✅ Always wrap in MaterialApp
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: MyCustomWidget(),
      ),
    ),
  );
  
  expect(find.byType(MyCustomWidget), findsOneWidget);
});
```

**For custom themes:**
```dart
await tester.pumpWidget(
  MaterialApp(
    theme: ThemeData(
      primarySwatch: Colors.blue,
      textTheme: customTextTheme,
    ),
    home: MyCustomWidget(),
  ),
);
```

---

#### 3. Finder Errors: Element Not Found

**Problem:** `expect(find.text('...'), findsOneWidget)` fails even though the text visibly exists.

**Common Causes:**

**Cause 1: Text wrapped in RichText/SelectableText**
```dart
// ❌ Won't find text inside RichText
expect(find.text('Hello'), findsOneWidget);

// ✅ Use descendant finders
expect(
  find.descendant(
    of: find.byType(RichText),
    matching: find.text('Hello'),
  ),
  findsOneWidget,
);
```

**Cause 2: Overflow widget hiding content**
```dart
// ✅ Provide explicit size constraints
await tester.pumpWidget(
  MaterialApp(
    home: SizedBox(
      width: 400,
      height: 600,
      child: MyWidget(),
    ),
  ),
);
```

**Cause 3: Animation/Transition in progress**
```dart
// ❌ Element not yet visible
await tester.tap(find.text('Next'));
expect(find.text('Page 2'), findsOneWidget); // FAILS

// ✅ Wait for animations
await tester.tap(find.text('Next'));
await tester.pumpAndSettle();
expect(find.text('Page 2'), findsOneWidget); // PASSES
```

---

### Golden Test Cross-Platform Reliability

#### 4. Font Rendering Inconsistencies

**Problem:** Golden tests pass locally but fail in CI with pixel differences in text.

**Root cause:** Different font engines, antialiasing, and default fonts across macOS, Linux, Windows, and Docker.

**✅ Best practice:** Use alchemist's CI variant (Ahem font) for CI. Keep platform baselines for local runs only. See the `flutter_test_config.dart` setup in the Basic Workflow.

#### 5. Pixel Difference Tolerance

Fix the source of the difference first (fonts, sizes, theme). If you still need a tolerance, write a custom `goldenFileComparator`, and check your alchemist version for its comparator API before you rely on it.

#### 6. Explicit Sizes

Give every `GoldenTestScenario` an explicit `SizedBox` size. Group related states with `GoldenTestGroup(children: [...])`.

#### 7. CI/CD Golden Test Best Practices

- Commit the CI baselines (`test/goldens/ci/`) and run `flutter test` in CI.
- Never run `flutter test --update-goldens` in CI. Update baselines locally and review the changed PNGs in the pull request.
- Fail the build on golden diffs.

### Mock Dependencies & Flaky Tests

#### 8. External Dependency Isolation

**Problem:** Tests are flaky due to real network calls, database queries, or file I/O.

**✅ Best Practice: Use Mocktail/Mockito with Dependency Injection**

**Example: Mocking API Client**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

// 1. Create mock
class MockApiClient extends Mock implements ApiClient {}
class FakeUser extends Fake implements User {}

void main() {
  setUpAll(() {
    // ✅ Register fallback for custom types
    registerFallbackValue(FakeUser());
  });

  group('UserRepository tests', () {
    late MockApiClient mockClient;
    late UserRepository repository;

    setUp(() {
      mockClient = MockApiClient();
      repository = UserRepository(apiClient: mockClient);
    });

    test('fetchUser returns user on success', () async {
      // 2. Stub the response
      final expectedUser = User(id: 1, name: 'Alice');
      when(() => mockClient.getUser(any()))
          .thenAnswer((_) async => expectedUser);

      // 3. Act
      final result = await repository.fetchUser(1);

      // 4. Assert
      expect(result, expectedUser);
      verify(() => mockClient.getUser(1)).called(1);
    });

    test('fetchUser throws on network error', () async {
      // ✅ Test error paths
      when(() => mockClient.getUser(any()))
          .thenThrow(NetworkException('Connection failed'));

      expect(
        () => repository.fetchUser(1),
        throwsA(isA<NetworkException>()),
      );
    });
  });
}
```

**Example: Mocking SharedPreferences**

```dart
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  setUp(() {
    // ✅ Use in-memory fake for SharedPreferences
    SharedPreferences.setMockInitialValues({
      'user_token': 'mock_token_12345',
      'theme_mode': 'dark',
    });
  });

  test('AuthService reads token correctly', () async {
    final prefs = await SharedPreferences.getInstance();
    final authService = AuthService(prefs);
    
    expect(authService.getToken(), 'mock_token_12345');
  });
}
```

---

#### 9. Async Test Timeouts

**Problem:** Tests hang or timeout waiting for async operations.

**Common Causes:**

**Cause 1: Infinite animations**
```dart
// ❌ pumpAndSettle hangs on CircularProgressIndicator
await tester.tap(find.text('Submit'));
await tester.pumpAndSettle(); // TIMEOUT after 10 minutes

// ✅ Use precise pump durations
await tester.tap(find.text('Submit'));
await tester.pump(); // Trigger loading state
await tester.pump(Duration(seconds: 2)); // Simulate API delay
expect(find.byType(CircularProgressIndicator), findsNothing);
```

**Cause 2: Real async without `runAsync`**
```dart
// ❌ Real file I/O not intercepted by FakeAsync
testWidgets('File upload', (tester) async {
  await tester.pumpWidget(MyApp());
  await tester.tap(find.text('Upload'));
  await tester.pumpAndSettle(); // Hangs - real I/O blocks
});

// ✅ Use runAsync for real system operations
testWidgets('File upload', (tester) async {
  await tester.pumpWidget(MyApp());
  
  await tester.runAsync(() async {
    // Escape FakeAsync boundary for real I/O
    await tester.tap(find.text('Upload'));
    await Future.delayed(Duration(seconds: 1)); // Real delay
  });
  
  await tester.pumpAndSettle();
});
```

---

#### 10. Memory Leaks in Tests

**⚠️ CRITICAL: "A Timer is still pending even after the widget tree was disposed"**

**Problem:** Tests fail with errors about pending Timers or uncanceled subscriptions.

**Root Cause:** Resources not cleaned up in widget `dispose()` method.

**❌ Anti-pattern:**
```dart
class MyWidget extends StatefulWidget {
  @override
  _MyWidgetState createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  Timer? _timer;
  
  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(Duration(seconds: 1), (_) {
      // Update state
    });
  }
  
  // ❌ No dispose - timer leaks!
}
```

**✅ Best Practice:**
```dart
class _MyWidgetState extends State<MyWidget> {
  Timer? _timer;
  
  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(Duration(seconds: 1), (_) {
      setState(() {/* update */});
    });
  }
  
  @override
  void dispose() {
    _timer?.cancel(); // ✅ Always cancel in dispose
    super.dispose();
  }
}
```

**Testing Timer-based widgets:**
```dart
testWidgets('Timer widget cleans up', (tester) async {
  await tester.pumpWidget(MaterialApp(home: TimerWidget()));
  
  // Advance virtual time
  await tester.pump(Duration(seconds: 5));
  
  // Dispose widget
  await tester.pumpWidget(SizedBox.shrink());
  
  // ✅ No pending timer error if dispose() implemented correctly
});
```

---

## Constraints

* **Always wrap test widgets in MaterialApp/CupertinoApp** unless explicitly testing widgets that don't require material/cupertino dependencies.
* **Use `loadAppFonts()` in CI/CD** to eliminate font rendering inconsistencies.
* **Mock all external dependencies** (network, database, file I/O) to avoid flaky tests.
* **Set explicit surface sizes** in golden tests for cross-platform consistency.
* **Never use `pumpAndSettle()` with infinite animations** - use `pump(Duration)` instead.
* **Always implement `dispose()`** for Timers, StreamSubscriptions, and AnimationControllers.
