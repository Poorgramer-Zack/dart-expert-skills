---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Advanced Testing Tools and Frameworks (Advanced Testing Tools)

## Goal
Implements advanced testing tools and frameworks to resolve common pain points in the native Flutter testing environment: dependency decoupling (Mocking), native system interaction (Native E2E), and test debugging (Test Debugging).

## Instructions

### Mockito (Mock Objects and Dependency Inversion)
[mockito](https://pub.dev/packages/mockito) is the most mainstream Mocking tool. When paired with `build_runner`, it can automatically generate fake classes.

#### 1.1 Installation and Configuration
```yaml
dependencies:
  mockito: ^5.8.1

dev_dependencies:
  build_runner: ^2.17.1
```

#### 1.2 Generating Mock Classes
Use the `@GenerateNiceMocks` annotation to generate Mock classes. The advantage of `NiceMocks` is that when it encounters an undefined method, it returns a safe null or default value instead of crashing.

```dart
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'api_client.dart';

// This will generate api_client.mocks.dart
@GenerateNiceMocks([MockSpec<ApiClient>()])
import 'api_client.mocks.dart';

void main() {
  test('Test API Call', () async {
    final mockClient = MockApiClient();
    
    // 🌟 Stubbing: Set the default return value when fetchUser is called
    when(mockClient.fetchUser(any))
        .thenAnswer((_) async => User(id: 1, name: 'Zack'));
        
    final repo = UserRepository(apiClient: mockClient);
    final user = await repo.getUser(1);
    
    expect(user.name, 'Zack');
    
    // 🌟 Verify the call count is as expected
    verify(mockClient.fetchUser(1)).called(1);
  });
}
```
Run `dart run build_runner build` to generate the files.

### Mocktail (Code-generation-free Mock Alternative)
[mocktail](https://pub.dev/packages/mocktail) is another highly popular choice comparable to `mockito`. Its greatest advantage is that **it absolutely does not rely on `build_runner` for code generation**, making it particularly suitable for developers who hate long build times.

#### 2.1 Installation and Basic Configuration
```yaml
dev_dependencies:
  mocktail: ^1.0.5
```

#### 2.2 Core Syntax and Usage
The underlying API of `mocktail` borrows heavily from `mockito`, but the implementation is different.

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

// 1. Directly declare a class that extends Mock and implements the target interface
class MockApiClient extends Mock implements ApiClient {}

// 🌟 Fallback for Custom Types (The core of Mocktail solving Null Safety)
class FakeUser extends Fake implements User {}

void main() {
  setUpAll(() {
    // If your method parameters use custom objects and any(), you MUST register the fallback in setUpAll
    registerFallbackValue(FakeUser());
  });

  test('Test using Mocktail', () async {
    final mockClient = MockApiClient();

    // 2. Stubbing (Identical to mockito)
    when(() => mockClient.fetchUser(any()))
        .thenAnswer((_) async => User(id: 2, name: 'Alice'));

    // 3. Act
    final repo = UserRepository(apiClient: mockClient);
    final user = await repo.getUser(2);

    // 4. Assert & Verify
    expect(user.name, 'Alice');
    verify(() => mockClient.fetchUser(2)).called(1);
  });
}
```

> **Comparison**:
> - **Mockito**: Requires `build_runner`. The advantage is all fallbacks and type-safety are handled automatically during the generation phase.
> - **Mocktail**: Write and test instantly, no generation wait time. The trade-off is manually writing many `Fake` classes to `registerFallbackValue()` when encountering `any()` with custom objects.

### Patrol (Native E2E Testing Framework) - 🌟 Industry Preferred E2E
The most fatal flaw of Flutter's native `integration_test` is the **inability to tap native system dialogs** (e.g., requesting location permissions, camera authorization, interacting with WebViews, push notifications). [Patrol](https://patrol.leancode.co/) solves this with a native automator bridge.

**Two packages — choose based on your needs:**
- `patrol` — full framework with native OS interaction, requires `patrol_cli` and platform setup
- `patrol_finders` — lightweight custom finder API only, works with standard `flutter test`, no native setup

#### 3.1 patrol_finders (Widget Test Enhancement — No Native Setup Required)

Drop-in enhancement for widget tests. Use `patrolWidgetTest` instead of `testWidgets`:

```yaml
dev_dependencies:
  patrol_finders: ^3.7.0
```

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:patrol_finders/patrol_finders.dart';

void main() {
  patrolWidgetTest('login flow', ($) async {
    await $.pumpWidgetAndSettle(const MyApp());

    // Chain selectors — far more readable than nested find.descendant()
    await $(#emailInput).enterText('user@test.com');
    await $(#passwordInput).enterText('secret');
    await $(Scaffold).$(#loginBox).$('Log in').tap();

    // Contextual scroll finders
    await $(Scrollable).containing(ElevatedButton).tap();

    expect($(Text).containing('Welcome'), findsOneWidget);
  });
}
```

Run with standard `flutter test` — no extra CLI needed.

#### 3.2 patrol (Full E2E with Native Interactions)

Use `patrolTest` for flows that cross the Flutter/OS boundary:

```yaml
dev_dependencies:
  patrol: ^4.11.0
```

```bash
# Install patrol CLI
dart pub global activate patrol_cli

# Run patrol tests (NOT flutter test)
patrol test --target integration_test/app_test.dart
```

```dart
import 'package:patrol/patrol.dart';

void main() {
  patrolTest(
    'notification permission flow',
    ($) async {
      await $.pumpWidgetAndSettle(const MyApp());

      await $(#enableNotifications).tap();

      // Platform API (patrol 4.0+) — replaces deprecated $.native.*
      await $.platform.mobile.grantPermissionWhenInUse();

      // Press Home, open notification tray, tap notification
      await $.platform.mobile.pressHome();
      await $.platform.mobile.openNotifications();
      await $.platform.tap(Selector(text: 'New message'));

      expect($('Chat'), findsOneWidget);
    },
  );
}
```

#### 3.3 Patrol 4.0 Platform API Reference

Patrol 4.0 reorganized native interactions into semantic categories. `$.native` is **deprecated**.

```dart
// Mobile (both Android + iOS)
await $.platform.mobile.pressHome();
await $.platform.mobile.pressBack();
await $.platform.mobile.openNotifications();
await $.platform.mobile.grantPermissionWhenInUse();
await $.platform.mobile.grantPermissionAlways();
await $.platform.mobile.denyPermission();
await $.platform.mobile.pressDoubleRecentApps();

// Android-specific
await $.platform.android.openNotifications();
await $.platform.android.tap(Selector(text: 'Allow'));

// iOS-specific
await $.platform.iOS.dismissSafariPrompt();

// Web
await $.platform.web.clearCookies();

// Cross-platform (same as old $.native at the top level)
await $.platform.openNotifications();
await $.platform.tap(Selector(text: 'OK'));
```

> **When to use each:**
> - `patrol_finders` — widget tests needing cleaner finder syntax, no CI complexity
> - `patrol` (full) — integration tests requiring permission dialogs, push notifications, WebViews, deep links

### Convenient Test (Enhancing Development and CI Experience)
[convenient_test](https://pub.dev/packages/convenient_test) is a heavy-duty testing plugin designed to solve the issues of "debugging difficulties" and "slow execution" in Flutter testing.

#### 4.1 Core Features
1. **Time Travel + Screenshots**: Automatically records video and captures screenshots for every step of your test. When a test fails, you can open the GUI and drag the timeline to see exactly what the screen looked like at that moment.
2. **5x+ Speed on Desktop**: Allows heavy Integration tests normally bound to mobile simulators to run at ultra-high speed on Host (Mac/Windows) desktop environments.
3. **Isolation Mode**: Ensures tests are completely isolated (via automatic Hot Restart), preventing a dirty state from the previous test affecting the next.
4. **Built-in Auto Retry Mechanism**: Say goodbye to endless `pumpAndSettle`. It continuously automatically tries to find elements before timing out.

#### 4.2 Basic Setup
```dart
import 'package:convenient_test/convenient_test.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  // Use tTestWidgets instead of testWidgets
  tTestWidgets('Tap button', (t) async {
    
    // No longer need to manually call pumpAndSettle, the framework automatically handles waiting!
    await t.get(ElevatedButton).tap();
    
    // Automatically generate Logs with friendly descriptions, displayed on the GUI panel
    t.log('Button tap concluded, preparing to verify');
    
    await t.get(find.text('Success')).should(findsOneWidget);
  });
}
```

> **Best Practice Workflow**:
> - Everyday small modules: Use `flutter_test` (Unit/Widget).
> - Core fool-proofing automation and debugging: Introduce `convenient_test` during daily development to accelerate verification.
> - Pre-submission/Final CI line of defense: Use `Patrol` to deploy the App onto Firebase Test Lab for brutal testing on real phones end-to-end (Including permission dialogs).

### Golden Tests (Visual Regression Testing)

Golden tests capture a widget as an image and compare it with a stored baseline, so unintended visual changes fail the build. Use them for design-system components, complex layouts, and cross-platform consistency.

> **Package:** use `alchemist`. `golden_toolkit` is discontinued. The steps below were verified with `alchemist` 0.14.1 and `flutter test`.

#### 5.1 Basic Golden Test Workflow

**Step 1: Add the dependency**
```yaml
dev_dependencies:
  alchemist: ^0.14.1
  flutter_test:
    sdk: flutter
```

**Step 2: Configure the test runner** (`test/flutter_test_config.dart`)
```dart
import 'dart:async';

import 'package:alchemist/alchemist.dart';

Future<void> testExecutable(FutureOr<void> Function() testMain) {
  return AlchemistConfig.runWithConfig(
    config: const AlchemistConfig(),
    run: () async => await testMain(),
  );
}
```

**Step 3: Write golden tests**
```dart
import 'package:alchemist/alchemist.dart';
import 'package:flutter/material.dart';

void main() {
  goldenTest(
    'renders a button in light theme',
    fileName: 'button_light',
    builder: () => GoldenTestGroup(
      children: [
        GoldenTestScenario(
          name: 'default',
          child: const SizedBox(
            width: 200,
            height: 50,
            child: ElevatedButton(onPressed: null, child: Text('Sign In')),
          ),
        ),
      ],
    ),
  );
}
```

**Step 4: Generate baselines, then run**
```bash
flutter test --update-goldens
flutter test
```

Alchemist writes one baseline per variant: `test/goldens/ci/` (Ahem font, for CI) and `test/goldens/<os>/` (the local platform font). Verified: both folders are created on the first `--update-goldens` run.

#### 5.2 Cross-Platform Reliability
- Font rendering differs between macOS, Linux, Windows, and Docker. The CI variant renders with the Ahem font, so CI results do not depend on system fonts.
- Keep the test surface explicit (an explicit `SizedBox` size in each scenario) so the output does not depend on the default test size.
- Commit the CI baselines. Generate platform baselines only on the matching OS.

#### 5.3 Several States in One Baseline
Put several `GoldenTestScenario` entries in one `GoldenTestGroup(children: [...])` to cover default, disabled, and loading states in a single test.

## Constraints
* Prefer `Mocktail` when rapid iteration speed is the top priority and manual fallback configuration is acceptable. Use `Mockito` when type-safety and automatic generation are preferred.
* Use `patrol_finders` (`patrolWidgetTest`) for widget tests that need cleaner finder syntax — no native setup required, runs with `flutter test`.
* Use full `patrol` package (with `patrol test` CLI) **only** when the test flow requires native OS interaction (permissions, push notifications, WebViews, system dialogs).
* **NEVER use `$.native.*`** — deprecated since Patrol 4.0. Use `$.platform.mobile.*`, `$.platform.android.*`, `$.platform.iOS.*`, or `$.platform.web.*` instead.
* **ALWAYS use `loadAppFonts()` in `test/flutter_test_config.dart`** when implementing golden tests to prevent font rendering inconsistencies.
* **Set explicit `surfaceSize`** in golden tests to ensure cross-platform pixel-perfect consistency.
* **Use tolerance thresholds (0.01-0.05)** for golden comparators to handle minor antialiasing differences across platforms.
