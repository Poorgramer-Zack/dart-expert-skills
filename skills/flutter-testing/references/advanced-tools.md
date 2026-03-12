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
  mockito: ^5.4.4

dev_dependencies:
  build_runner: ^2.4.0
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
  mocktail: ^1.0.4
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
The most fatal flaw of Flutter's native `integration_test` is the **inability to tap native system dialogs** (e.g., requesting location permissions, camera authorization, interacting with WebViews, push notifications, etc.). [Patrol](https://patrol.leancode.co/) perfectly solves this problem.

#### 3.1 Core Advantages
1. **Crossing Native Boundaries**: Can interact with iOS/Android system UI via Dart code.
2. **More Concise Custom Finders**: Significantly reduces test code verbosity (using syntax like `$()`).
3. **Device Farm Support**: Perfectly supports cloud device farm testing like Firebase Test Lab.

#### 3.2 Syntax Comparison and Usage
Use `patrolTest` instead of `testWidgets`:

```dart
import 'package:patrol/patrol.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/main.dart' as app;

void main() {
  patrolTest(
    'Complete registration flow and accept system permission prompt',
    ($) async { // 🌟 Notice the special "$" symbol; this is the PatrolFinder
      
      app.main();
      await $.pumpAndSettle();

      // Extremely concise tap and enter text syntax
      await $(#emailTextField).enterText('test@email.com');
      await $('Login').tap();

      // 🌟🌟 Killer feature: Interacting with native system dialogs!
      if (await $.native.isPermissionDialogVisible()) {
        await $.native.grantPermissionWhenInUse(); // Taps the system's "Allow"
      }
      
      // Verify push notification appearance
      await $.native.openNotifications();
      // ...
    },
  );
}
```

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

## Constraints
* Prefer `Mocktail` when rapid iteration speed is the top priority and manual fallback configuration is acceptable. Use `Mockito` when type-safety and automatic generation are preferred.
* Utilize `Patrol` for CI environments specifically whenever there are system-level components (location, permissions, webviews, notification trays) involved in user flows.
