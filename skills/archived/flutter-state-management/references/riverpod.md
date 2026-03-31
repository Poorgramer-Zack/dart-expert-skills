---
name: "riverpod-best-practices"
description: "Best practices and guidelines for Riverpod Latest Version Best Practices Guide (v3.1.x / v3.3.x). Use this skill when you need to write or review code related to Riverpod Latest Version Best Practices Guide (v3.1.x / v3.3.x)."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Riverpod Latest Version Best Practices Guide (v3.2.x)

## Goal
Riverpod is an extremely popular reactive state management and dependency injection framework in the Flutter ecosystem. Currently, the latest version is roughly **3.3.x**.

This guide covers two best practice methodologies: **Code Generation-based (Officially Highly Recommended)** and **Traditional Manual approaches**, deeply dissecting Riverpod 3.0's new features alongside advanced application patterns for various Providers.

## Instructions

### 1. Riverpod 3.0 Killer Features: Auto Retry & More
Riverpod 3.0 introduces profound architectural improvements and API simplifications, notably fortifying fault tolerance against unstable network conditions.

#### 1.1 Automatic Retry Mechanism
Previously, a failed `FutureProvider` instantly crashed into an Error state. Now, Riverpod embeds an **Exponential Backoff** auto-retry mechanism.
*   **Default Behavior**: Initialization failures autonomously delay and retry (default max 10 attempts, spanning 200ms extending to 6.4s conditionally).
*   **UX Enhancements**: Sustains UI rendering within a "blink and recover" matrix rather than flashing red error screens. Providers preserve `Loading` states throughout retry sequences natively.
*   **Customization**: Alter parameters directly inside `@Riverpod` annotations or via global ProviderScope constants:

```dart
// Defining a custom retry strategy (e.g., maximum 5 retries with exponential backoff)
Duration? myRetry(int retryCount, Object error) {
  if (retryCount >= 5) return null; // Return null to stop retrying
  return Duration(milliseconds: 200 * (1 << retryCount));
}

// Applying the custom retry configuration uniquely isolating specified Providers
@Riverpod(retry: myRetry)
Future<UserProfile> userProfile(Ref ref) async {
  return fetchProfile();
}

// Disabling retry mechanics entirely (returning null instantly)
@Riverpod(retry: (_, __) => null)
Future<UserProfile> userProfileDisabled(Ref ref) async {
  return fetchProfile();
}
```

#### 1.2 Provider Auto-Pause (Out-of-view Pausing)
When actively tracked Widgets exit immediately visible screen bounds (e.g., pushed into backgrounds awaiting pop, avoiding premature disposal), Riverpod 3.0 intelligently **pauses** computation recalculations for that Provider by default until visibility restores (**resumes**), drastically conserving Memory and CPU overheads universally.
*   **Preventing Pausing via TickerMode**: If you require a Provider to continue listening and reacting even when its `Consumer` is completely off-screen, you must explicitly wrap that Consumer tree with `TickerMode(enabled: true)`.

#### 1.3 Phenomenal API Simplifications
*   `AutoDispose` base interfaces entirely **Removed/Unified**. Instead of differentiating `AutoDisposeNotifier` vs `Notifier`, Riverpod 3.0 combines them natively into `Notifier` / `AsyncNotifier` structures outright simplifying architectural blueprints radically.
*   `Family` base interfaces **Removed**. `FamilyNotifier` is obsolete. Standard `Notifier` bases simply require constructor arguments directly to consume family traits securely natively.
*   `Ref` objects eliminate convoluted generic requirements (ditching `Ref<NotifierProvider>`), generating drastically cleaner syntactic layouts inherently operating as `Ref`.
*   Introduces **`ref.mounted`**, definitively verifying Widget or Provider lifecycle survival sequences following asynchronous executions averting `StateError` anomalies entirely.

```dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  bool build() => false;

  Future<void> login() async {
    await apiLoging(); // Arduous execution delay
    if (!ref.mounted) return; // 🌟 3.0 Nascent Safety Net Implementation!
    state = true;
  }
}
```

### 2. Deep Dive: Riverpod Provider Types Paradigm

#### 2.1 Read-Only Data Collectives (Data Fetching)
Engineered exclusively provisioning immutable configurations reacting identically mirroring pure `StatelessWidget` architectures.
*   **`Provider`**: Supplying static constants, Repository singletons, or purely Computed synchronous derivatives reacting explicitly tracking alternative Providers.
*   **`FutureProvider`**: Encapsulating singular asynchronous executions (e.g. API demands). UI naturally extracts corresponding `loading/data/error` structures leveraging native `AsyncValue` outputs seamlessly.
*   **`StreamProvider`**: Ceaselessly monitoring dynamic continuous inputs (Firebase DBs, WebSockets, persistent Location listeners).

#### 2.2 Mutable State Consortia (Business Logic)
Encapsulating complex internal methodologies explicitly granting external forces mutation authority. Replicating `StatefulWidget` operational blueprints fundamentally.
*   **`StateProvider` / `StateNotifierProvider` / `ChangeNotifierProvider` (Legacy Phasing)**: Conditionally tolerated exclusively warehousing absolute minimalist primitives (String, Enum, Boolean). *⚠️ Post-3.0 architectures strongly mandate sweeping these towards NotifierProviders to enforce rigorous protective encapsulations universally. They are now officially grouped under legacy imports requiring explicit inclusion `import 'package:flutter_riverpod/legacy.dart';` to operate.*
*   **`NotifierProvider` (Modern Standard Synchronization Master)**: Annihilates legacy `StateNotifierProvider` methodologies completely. Consolidates intricate synchronous modifications tightly confining robust logical methods dynamically deploying internal `ref.read`/`ref.watch` cross-tracking interrogations unrestrictedly.
*   **`AsyncNotifierProvider` (Modern Asynchronous Control System)**: Advanced extension dominating asynchronous `Future` initialization demands intricately paired deploying successive subsequent asynchronous modifying maneuvers (e.g. fetching TodoLists initially anticipating subsequent remote server-side deletion executions later).

### 3. Core Universal Best Practices (Applicable Across All Syntactic Topologies)

1. **Prioritize Feature-First Geographical Organization**: Station underlying localized Providers tightly juxtaposed explicitly bordering interacting isolated targeted UI sectors, dismantling universally centralized `providers.dart` file blackholes stringently.
2. **Absolute Discipline Negotiating `ref.watch` vs `ref.read`**:
   * Penetrating interior generic Widget `build` frames or internal central Provider bodies, **ETERNALLY DEPLOY `ref.watch`** anchoring explicit systemic recalculation dependencies universally.
   * Trafficking embedded callbacks (`onPressed`) or traversing unrepeated static lifecycles (`initState`), **INFALLIBLY DEPLOY `ref.read`** executing one-shot isolated functional triggers devoid lingering reactive anchors definitively.
3. **Rigorous Value Equality Enforcement (`==` & `hashCode`)**: Riverpod inherently relies precisely tracking identical value transitions manipulating object comparisons natively. Conclusively deploy explicit Freezed or Equatable packages enforcing absolute model-layer equivalence standards unequivocally.

### 4. Implementation 1: Code Generation Pattern (Officially Heavily Championed)
Unambiguously transitioning freshly minted architectures leveraging comprehensive `riverpod_generator` coupled embracing `@riverpod` macros universally.

#### Dependencies
```yaml
dependencies:
  flutter_riverpod: ^3.3.0
  riverpod_annotation: ^4.0.2

dev_dependencies:
  build_runner: ^2.4.0
  riverpod_generator: ^3.0.0
  custom_lint: ^0.6.0
  riverpod_lint: ^3.0.0 # Breathtakingly exceptional furnishing precise auto-fixes and comprehensive preventative fool-proofing diagnostic inspections natively.
```

#### Synchronous & Asynchronous Generations (Provider / FutureProvider)
Dismiss manual classifications; simply append `@riverpod` directly atop generic functions immediately!

```dart
part 'user_provider.g.dart';

// Synchronous Provider Definition
@riverpod
Dio dioClient(Ref ref) => Dio(BaseOptions(baseUrl: 'https://api.com'));

// Asynchronous FutureProvider Construction
@riverpod
Future<String> fetchUserData(Ref ref, {required int userId}) async {
  final dio = ref.watch(dioClientProvider);
  final response = await dio.get('/users/$userId');
  return response.data['name'];
}
```

#### Interactive Manipulations (NotifierProvider / AsyncNotifierProvider)
Construct custom subclasses inheriting dynamically parsed `_$ClassName` directives:

```dart
// Base Notifier generation seamlessly finalized inherently without AutoDispose prefixing
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++; // Effortlessly notifies responsive UI bindings automatically
}

// ⚠️ AsyncNotifier Blueprint Paradigm & ProviderException Catching
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<String>> build() async => _fetchFromServer();

  Future<void> addTodo(String todo) async {
    state = const AsyncValue.loading(); // Retaining archived local cache momentarily exhibiting loading states
    
    // Deploying AsyncValue.guard handling elaborate exhaustive try/catch blocks flawlessly encapsulating execution anomalies defensively
    state = await AsyncValue.guard(() async {
      await _addToServer(todo);
      return _fetchFromServer(); // Synthetically recalculating freshly archived remote server pools inherently
    });
  }
}

// Interacting & Catching explicit `ProviderException` safely
Future<void> userTriggeredAction() async {
  try {
     // Triggering operations. 
     // Note: If the backend build architecture failed fundamentally, Riverpod 3.0 wraps every failing error neatly inside \`ProviderException\`.
     await ref.read(todoListProvider.future);
  } on ProviderException catch(e) {
     if(e.exception is CustomNetworkException) {
         // handle
     }
  }
}
```

### 5. Implementation 2: Traditional Manual Syntax Framework
Strictly rejecting code generation dependencies dictates utterly abandoning legacy `StateProvider`/`StateNotifier` classes natively shifting operations entirely exploiting unified **`Notifier`** partnered **`AsyncNotifier`** base derivatives fundamentally (Recall: AutoDispose explicitly integrated cleanly inside the base).

```dart
// 1. Fundamental FutureProvider deployment (No AutoDispose prefix needed magically)
final userDataProvider = FutureProvider.family<String, int>((ref, userId) async {
  return fetchUser(userId);
});

// 2. Refined NotifierProvider manual standard
class CounterNotifier extends Notifier<int> {
  // Family Traits simply ingested via standard constructor arguments cleanly rather than FamilyNotifier nesting!
  CounterNotifier([this.startingValue = 0]);
  final int startingValue;

  @override
  int build() => startingValue;
  
  void increment() => state++;
}

final counterProvider = NotifierProvider<CounterNotifier, int>(
  CounterNotifier.new, 
);
```

### 6. Riverpod Testing Best Practices (3.0 Standards)
Thanks to its architectural design (`ProviderContainer`), Riverpod is inherently extremely suitable for testing, and dependencies can be highly replaced **without even needing** mocktail/mockito for basic providers.

#### 6.1 Core Principle: Unit Testing with `ProviderContainer.test()`
In unit testing, we instantiate a `ProviderContainer` directly, allowing us to easily override internal states natively. *Note: Riverpod 3.0 recommends using `ProviderContainer.test()` specifically for isolated test environments.*

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Assuming we have these Providers
@riverpod
Api api(Ref ref) => RealApi();

@riverpod
Future<User> user(Ref ref) async => ref.watch(apiProvider).fetchUser();

void main() {
  test('Test userProvider returns Mock data', () async {
    // 1. Arrange: Instantiate Container for tests and pass in overrides
    final container = ProviderContainer.test(
      overrides: [
        // 🌟 Directly replace the underlying apiProvider; other providers above relying on this API will automatically get the Mock value
        apiProvider.overrideWithValue(MockApi()), 
      ],
    );
    
    // Always remember to tear down at the end of the test to release resources
    addTearDown(container.dispose);

    // 2. Act: Directly read the Provide
    final user = await container.read(userProvider.future);

    // 3. Assert
    expect(user.name, 'Mock User');
  });
}
```

#### 6.2 Overrides in Widget Tests
When conducting UI (Widget) tests, leverage `tester.container()` to interact with providers, and directly override within the `ProviderScope`.

```dart
testWidgets('Test the homepage renders Mock data', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        // Override providers that rely on external or network dependencies natively
        authProvider.overrideWith((ref) => MockAuth()),
      ],
      child: const MaterialApp(home: HomePage()),
    ),
  );

  // Read provider from the active testing container
  final container = tester.container();
  expect(container.read(authProvider), 'MockAuth');
});
```

#### 6.3 ⚠️ Avoid Mocking Notifiers Directly
It is **strongly discouraged** to mock Notifiers directly using Mockito/Mocktail interfaces because Notifiers rely on complex internal lifecycle infrastructures.
Instead, abstract the data source (e.g. mock the Repository provider the Notifier reads).
If you absolutely *must* mock a Notifier explicitly, you **must subclass** its generated base class, otherwise you will break the interface:

```dart
// CORRECT approach to mocking a code-generated Notifier:
class MyNotifierMock extends _$MyNotifier with Mock implements MyNotifier {}
```


### 7. Riverpod 3.0 Experimental Features: Mutations & Offline Persistence
Riverpod 3.0 introduces highly anticipated core mechanisms previously demanding tedious manual wiring or third-party packages.

#### 7.1 Mutations (Asynchronous Operations & Feedback)
`Mutation<T>` drastically simplifies tracking arbitrary asynchronous side-effects (like submitting a form) outside standard Provider caching, granting effortless visual feedback (Pending, Error).

```dart
// 1. Typically defined globally or statically inside a Notifier
final addTodo = Mutation<Todo>();

// 2. Consumed gracefully inside the UI extracting precise status states
class TodoButton extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Reactively tracking the pending/error footprint
    final addTodoState = ref.watch(addTodo);
    
    return ElevatedButton(
      onPressed: () {
        // Run the mutation sequence isolated
        addTodo.run(ref, (tsx) async {
           await repository.uploadNewTodo();
        });
      },
      child: addTodoState is MutationPending 
          ? const CircularProgressIndicator() 
          : const Text('Add Todo'),
    );
  }
}
```

#### 7.2 Offline Persistence (State Caching built-in)
Persisting simple Provider configurations locally previously required Hive/SharedPreferences. Riverpod 3.0 natively introduces `Storage` gateways and the `.persist()` directive enabling automated cache-recoveries instantaneously triggering off the Notifier's `build` sequences.

```dart
import 'package:flutter_riverpod/experimental/persist.dart';

// 1. Defining the Database Gateway (e.g., using riverpod_sqflite)
@riverpod
Future<Storage<String, String>> storage(Ref ref) async {
  return JsonSqFliteStorage.open(join(await getDatabasesPath(), 'riverpod.db'));
}

// 2. Effortless persistent loading hooked sequentially into the build cycle
@riverpod
class Settings extends _$Settings {
  @override
  Future<ThemeMode> build() async {
    // Instruct Riverpod to intercept, decode, and cache updates automatically
    persist(
      ref.watch(storageProvider.future),
      key: 'user_theme',
      encode: (theme) => theme.name,
      decode: (json) => ThemeMode.values.byName(json as String),
    );

    return ThemeMode.system; // Fallback fallback execution
  }
}
```

### 8. Summary Conclusion
*   **Wholesale transition enforcing external state management protocols extracting dependencies distinctly removed from rigid native Widget hierarchical chains completely**.
*   Vigorously champion embracing prevailing **Implementation 1 (Code Generation)** architectures explicitly aligning harmonizing alongside ascending modern Dart 3 systemic constraints universally securely effectively unlocking full ecosystem utility features leveraging embedded `riverpod_lint` tooling completely securely natively flawlessly!

## Constraints
* Assure `.listen` calls strictly avoid internal recursive state modifications inherently risking generating infinite rebuild chain-reaction collapses unconditionally. 
* Strictly eradicate referencing native UI internal structures (such as `BuildContext`) traversing any Provider bodies entirely uniformly comprehensively!
* Leverage `ProviderContainer` to interact directly with Riverpod `Provider` states without the need to mount a complete Widget Tree for logic tests.
