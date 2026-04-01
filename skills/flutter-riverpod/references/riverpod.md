---
name: "riverpod-patterns"
description: "Reference guide for Riverpod v3.3.x patterns. Covers code generation with @riverpod, manual Notifier/AsyncNotifier providers, unit and widget testing with ProviderContainer.test(), and experimental Mutations and offline persistence features."
metadata:
  last_modified: "2026-04-01 15:50:00 (GMT+8)"
---

# Riverpod v3.3.x Patterns

## Table of Contents

- [New in v3.0](#new-in-v30)
- [Consumer vs ConsumerWidget](#consumer-vs-consumerwidget)
- [Provider Types](#provider-types)
- [Code Generation Patterns](#code-generation-patterns)
  - [Generic Providers](#generic-providers-code-generation-only)
  - [Statically Safe Scoping](#statically-safe-scoping-code-generation-only)
  - [Catching ProviderException](#catching-providerexception)
- [Manual Patterns](#manual-patterns)
- [Testing](#testing)
  - [overrideWithBuild](#overridewithbuild--mock-only-the-initial-state)
  - [overrideWithValue (restored)](#futureprovideroverridewithvalue-restored)
- [Experimental Features](#experimental-features)
- [Constraints](#constraints)

---

## New in v3.0

### Auto-Retry

Failed `FutureProvider` now auto-retries with exponential backoff (default: 10 attempts, 200ms–6.4s). UI stays in `loading` state during retries rather than flashing an error.

```dart
Duration? myRetry(int retryCount, Object error) {
  if (retryCount >= 5) return null; // stop retrying
  return Duration(milliseconds: 200 * (1 << retryCount));
}

@Riverpod(retry: myRetry)
Future<UserProfile> userProfile(Ref ref) async => fetchProfile();

// Disable retry entirely
@Riverpod(retry: (_, __) => null)
Future<UserProfile> noRetryProfile(Ref ref) async => fetchProfile();
```

### Provider Auto-Pause

Off-screen providers pause computation automatically to save CPU/memory. To keep a provider active when its widget is off-screen:

```dart
TickerMode(enabled: true, child: Consumer(...))
```

### API Simplifications

- `AutoDisposeNotifier` and `Notifier` unified — no longer separate classes.
- `FamilyNotifier` removed — pass constructor arguments directly to `Notifier`.
- `Ref` no longer requires generic type parameters; all `Ref` subclasses (`FutureProviderRef`, etc.) removed — use `Ref` directly.
- `ref.listenSelf` and `ProviderRef.state` moved into `Notifier`/`AsyncNotifier` as instance members.
- `ref.mounted` added to safely check provider lifecycle after async operations.
- Legacy `StateProvider`, `StateNotifierProvider`, `ChangeNotifierProvider` moved to `package:flutter_riverpod/legacy.dart`.
- All lifecycle listeners (`ref.onDispose`, `ref.onCancel`, etc.) now **return a removal function** — call it to unsubscribe early:

```dart
final removeListener = ref.onDispose(() => print('disposed'));
removeListener(); // unsubscribe before provider is disposed
```

### Weak Listeners

`ref.listen` now accepts `weak: true` — the listened provider can still be auto-disposed even while being listened to. Use when combining multiple "sources of truth" without forcing a provider to stay alive:

```dart
ref.listen(anotherProvider, weak: true, (previous, next) {
  // callback fires when anotherProvider changes,
  // but does not prevent anotherProvider from auto-disposing
});
```

### AsyncValue Improvements

`AsyncValue` is now **sealed** — use exhaustive pattern matching without a `default` clause:

```dart
switch (ref.watch(userProvider)) {
  case AsyncData(:final value):
    return Text(value.name);
  case AsyncError(:final error):
    return Text('Error: $error');
  case AsyncLoading(:final progress):
    return LinearProgressIndicator(value: progress); // optional progress 0.0–1.0
}
```

Report upload progress from a Notifier:

```dart
class UploadNotifier extends AsyncNotifier<void> {
  @override
  Future<void> build() async {}

  Future<void> upload(File file) async {
    state = AsyncLoading(progress: 0.0);
    await uploadChunk(file, onProgress: (p) {
      state = AsyncLoading(progress: p);
    });
    state = const AsyncData(null);
  }
}
```

### Breaking: `==` Equality for All Providers

v3.0 unified state comparison to always use `==` (not `identical`). Previously `Provider` used `identical` while `StateProvider` used `==`, causing inconsistency. **Impact**: if your state objects don't implement `==` correctly, you may see extra or missed rebuilds. Fix: use `@freezed` for complex state objects or implement `operator ==` / `hashCode` manually.

```dart
// Bad: same content, different instance → may unexpectedly NOT rebuild in v2, WILL rebuild in v3
state = User(name: 'Alice'); // new instance each time

// Good: Freezed handles == correctly
@freezed
class User with _$User {
  factory User({required String name}) = _User;
}
```

### Breaking: Notifier Regeneration

From v3.0, `Notifier`/`AsyncNotifier` creates a **fresh instance on every rebuild**. Resources (timers, controllers, stream subscriptions) stored directly inside the Notifier will **leak**. Extract them into providers and bind lifecycle with `ref.onDispose`.

```dart
// Bad (v2 style) — timer leaks on rebuild
class PollingNotifier extends Notifier<int> {
  late Timer _timer;
  @override
  int build() {
    _timer = Timer.periodic(Duration(seconds: 1), (_) => state++); // leaks!
    return 0;
  }
}

// Good (v3 style) — extract resource to a provider
final timerProvider = Provider<Timer>((ref) {
  final timer = Timer.periodic(Duration(seconds: 1), (_) {});
  ref.onDispose(timer.cancel);
  return timer;
});

class PollingNotifier extends Notifier<int> {
  @override
  int build() {
    ref.watch(timerProvider); // lifecycle managed externally
    return 0;
  }
}
```

### FamilyNotifier Migration

```dart
// v2 — FamilyNotifier with build(arg)
class CounterNotifier extends FamilyNotifier<int, String> {
  @override
  int build(String arg) => 0;
}

// v3 — pass arg via constructor
class CounterNotifier extends Notifier<int> {
  CounterNotifier(this.arg);
  final String arg;
  @override
  int build() => 0;
}
final provider = NotifierProvider.family<CounterNotifier, int, String>(CounterNotifier.new);
```

---

## Consumer vs ConsumerWidget

Prefer `Consumer` over `ConsumerWidget` to minimise rebuild scope. `ConsumerWidget` rebuilds the **entire widget subtree** when any watched provider changes. `Consumer` scopes the rebuild to only the wrapped builder function.

### Decision Table

| Scenario | Use |
|---|---|
| Entire widget tree depends on the same provider | `ConsumerWidget` |
| Only part of the tree needs provider data | `Consumer` (scoped) |
| Mixing with `StatefulWidget` lifecycle | `ConsumerStatefulWidget` |
| Combining with `flutter_hooks` | `HookConsumerWidget` |
| Rebuild only on a specific field of a large state | `ref.watch(provider.select(...))` |

### Scoped Consumer Pattern

```dart
// Bad: entire Scaffold rebuilds when userName changes
class ProfileScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userName = ref.watch(userProvider.select((u) => u.name));
    return Scaffold(
      appBar: AppBar(title: Text(userName)), // only this needs provider
      body: const ExpensiveStaticContent(),  // unnecessarily rebuilt
    );
  }
}

// Good: only AppBar rebuilds; ExpensiveStaticContent is untouched
class ProfileScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Consumer(
          builder: (context, ref, _) {
            final userName = ref.watch(userProvider.select((u) => u.name));
            return Text(userName);
          },
        ),
      ),
      body: const ExpensiveStaticContent(),
    );
  }
}
```

### `select()` for Field-Level Precision

```dart
Consumer(
  builder: (context, ref, _) {
    // Rebuilds only when isLoggedIn changes, not on any User field change
    final isLoggedIn = ref.watch(authProvider.select((u) => u.isLoggedIn));
    return isLoggedIn ? const HomeView() : const LoginView();
  },
)
```

### `ref.listen` for Side Effects Without Rebuild

```dart
// In build() — no rebuild, just triggers a callback
ref.listen<AsyncValue<void>>(saveProvider, (prev, next) {
  if (next is AsyncError) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Save failed: ${next.error}')),
    );
  }
});
```

---

## Provider Types

| Provider | Use Case |
|----------|----------|
| `Provider` | Static values, repository singletons, computed sync values |
| `FutureProvider` | Single async fetch (API call) |
| `StreamProvider` | Continuous data (Firebase, WebSocket, GPS) |
| `NotifierProvider` | Synchronous mutable state with methods |
| `AsyncNotifierProvider` | Async mutable state with methods |
| `StateProvider` | Simple primitive state — legacy, avoid in new code |

---

## Code Generation Patterns

```yaml
dev_dependencies:
  riverpod_generator: ^3.3.1
  custom_lint: ^0.6.0
  riverpod_lint: ^3.0.0 # provides auto-fixes and diagnostic inspections
```

### Provider and FutureProvider

```dart
part 'user_provider.g.dart';

@riverpod
Dio dioClient(Ref ref) => Dio(BaseOptions(baseUrl: 'https://api.com'));

@riverpod
Future<String> fetchUserData(Ref ref, {required int userId}) async {
  final dio = ref.watch(dioClientProvider);
  final response = await dio.get('/users/$userId');
  return response.data['name'] as String;
}
```

### NotifierProvider

```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
}
```

### AsyncNotifierProvider

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<String>> build() async => _fetchFromServer();

  Future<void> addTodo(String todo) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      await _addToServer(todo);
      return _fetchFromServer();
    });
  }
}
```

### Generic Providers (code-generation only)

Type parameters work like additional family arguments:

```dart
@riverpod
T multiply<T extends num>(Ref ref, T a, T b) => a * b;

// Usage
final intResult = ref.watch(multiplyProvider<int>(2, 3));       // 6
final doubleResult = ref.watch(multiplyProvider<double>(2.5, 3.5)); // 8.75
```

### Statically Safe Scoping (code-generation only)

Mark a provider as "scoped" with `dependencies: []` — `riverpod_lint` will error if any usage site forgets to override it:

```dart
// Declare a scoped provider (must be overridden before use)
@Riverpod(dependencies: [])
Future<int> currentUserId(Ref ref) => throw UnimplementedError();
```

**Option A** — override via `ProviderScope` at the usage site:

```dart
ProviderScope(
  overrides: [currentUserIdProvider.overrideWithValue(AsyncValue.data(42))],
  child: Consumer(
    builder: (context, ref, _) {
      final id = ref.watch(currentUserIdProvider);
      return Text('$id');
    },
  ),
)
```

**Option B** — propagate dependency with `@Dependencies`:

```dart
@Dependencies([currentUserId])
class UserCard extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final id = ref.watch(currentUserIdProvider);
    return Text('$id');
  }
}
```

Using `@Dependencies` shifts the override requirement up the widget tree — `riverpod_lint` traces it statically.

### Catching ProviderException

When a provider's `build` throws, Riverpod 3.0 wraps it in `ProviderException`.

```dart
Future<void> userAction(WidgetRef ref) async {
  try {
    await ref.read(todoListProvider.future);
  } on ProviderException catch (e) {
    if (e.exception is NetworkException) {
      // handle
    }
  }
}
```

---

## Manual Patterns

Avoids code generation. Uses `Notifier` and `AsyncNotifier` directly — auto-dispose is built in, no prefix needed.

```dart
// FutureProvider with family
final userDataProvider = FutureProvider.family<String, int>((ref, userId) async {
  return fetchUser(userId);
});

// NotifierProvider
class CounterNotifier extends Notifier<int> {
  CounterNotifier([this.startingValue = 0]);
  final int startingValue;

  @override
  int build() => startingValue;

  void increment() => state++;
}

final counterProvider = NotifierProvider<CounterNotifier, int>(CounterNotifier.new);
```

---

## Testing

Riverpod's `ProviderContainer` design makes providers testable without mock frameworks for basic cases.

### Unit Tests with `ProviderContainer.test()`

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

@riverpod
Api api(Ref ref) => RealApi();

@riverpod
Future<User> user(Ref ref) async => ref.watch(apiProvider).fetchUser();

void main() {
  test('userProvider returns mock data', () async {
    final container = ProviderContainer.test(
      overrides: [apiProvider.overrideWithValue(MockApi())],
    );
    addTearDown(container.dispose);

    final user = await container.read(userProvider.future);
    expect(user.name, 'Mock User');
  });
}
```

### Widget Tests

```dart
testWidgets('homepage renders mock data', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [authProvider.overrideWith((ref) => MockAuth())],
      child: const MaterialApp(home: HomePage()),
    ),
  );

  final container = tester.container();
  expect(container.read(authProvider), isA<MockAuth>());
});
```

### Mocking Notifiers

Never mock a Notifier via Mockito directly — it breaks the internal lifecycle. Subclass the generated base class instead:

```dart
class MyNotifierMock extends _$MyNotifier with Mock implements MyNotifier {}
```

Prefer mocking the repository/data source the Notifier reads over mocking the Notifier itself.

### `overrideWithBuild` — Mock Only the Initial State

Overrides only the `build()` method, keeping all other Notifier methods intact:

```dart
test('counter starts at 42', () {
  final container = ProviderContainer.test(
    overrides: [
      counterProvider.overrideWithBuild((ref) => 42), // increment() still works
    ],
  );

  expect(container.read(counterProvider), 42);
  container.read(counterProvider.notifier).increment();
  expect(container.read(counterProvider), 43);
});
```

### `FutureProvider.overrideWithValue` (restored)

Previously removed, now back — initialise a `FutureProvider` or `StreamProvider` with a specific `AsyncValue` in tests:

```dart
final container = ProviderContainer.test(
  overrides: [
    userProvider.overrideWithValue(AsyncValue.data(User(name: 'Mock'))),
    streamProvider.overrideWithValue(AsyncValue.loading()),
  ],
);

---

## Experimental Features

> ⚠️ APIs in this section are experimental. Usable in production, but may change in breaking ways without a major version bump.

### Mutations

Solves two problems:
1. **Side-effect feedback**: show loading/success/error state for a button click or form submit without polluting provider state.
2. **Auto-dispose race condition**: `tsx.get()` keeps a provider alive until the mutation completes, preventing disposal mid-flight.

Declare as a top-level variable like a provider:

```dart
final addTodoMutation = Mutation<void>();
```

Watch the mutation state for exhaustive UI handling:

```dart
class AddTodoButton extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return switch (ref.watch(addTodoMutation)) {
      MutationIdle() => ElevatedButton(
          onPressed: () => addTodoMutation.run(ref, (tsx) async {
            // Use tsx.get() instead of ref.read() — keeps the provider alive
            // until the mutation finishes (prevents auto-dispose race condition)
            await tsx.get(todoListProvider.notifier).addTodo('New Todo');
          }),
          child: const Text('Submit'),
        ),
      MutationPending() => const CircularProgressIndicator(),
      MutationError(:final error) => Column(children: [
          Text('Failed: $error'),
          ElevatedButton(
            onPressed: () => addTodoMutation.run(ref, (tsx) async {
              await tsx.get(todoListProvider.notifier).addTodo('New Todo');
            }),
            child: const Text('Retry'),
          ),
        ]),
      MutationSuccess() => const Text('Added!'),
    };
  }
}
```

### Offline Persistence

Riverpod provides **interfaces only** — it does not bundle a database. Add `riverpod_sqflite` for the official SQLite adapter:

```yaml
dependencies:
  riverpod_sqflite: ^0.1.0  # official SQLite storage adapter
```

Call `persist()` at the start of `build()`. Riverpod will:
- **On first load**: immediately restore the cached value while the network fetch is in flight
- **After fetch**: server data takes precedence and is written back to cache
- **On state mutation**: automatically persists new state — no extra logic needed

```dart
import 'package:flutter_riverpod/experimental/persist.dart';
import 'package:riverpod_sqflite/riverpod_sqflite.dart';

// Share one Storage instance across all providers
final storageProvider = FutureProvider<JsonSqFliteStorage>((ref) async {
  return JsonSqFliteStorage.open(
    join(await getDatabasesPath(), 'riverpod.db'),
  );
});

@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    persist(
      ref.watch(storageProvider.future), // no need to await
      key: 'todos',                      // unique key per provider
      encode: jsonEncode,
      decode: (json) => (jsonDecode(json) as List)
          .map((e) => Todo.fromJson(e as Map<String, Object?>))
          .toList(),
      // Optional: default is 2 days; use unsafe_forever for permanent cache
      // options: const StorageOptions(cacheTime: StorageCacheTime.unsafe_forever),
    );

    // Cached value is shown immediately; server response replaces it when ready
    return fetchTodos();
  }

  Future<void> add(Todo todo) async {
    // No extra persist logic needed — Riverpod auto-caches on state change
    state = AsyncData([...await future, todo]);
  }
}
```

**Cache duration options:**

| Option | Behaviour |
|---|---|
| *(default)* | Cache valid for 2 days |
| `StorageCacheTime.unsafe_forever` | Never expires — use for user preferences, not server data |

---

## Constraints

- Never reference `BuildContext` inside any provider body.
- Avoid `ref.watch` inside `.listen` callbacks — risk of infinite rebuild loops.
- Use `ProviderContainer` directly for logic tests; no widget tree required.
- Co-locate providers with their feature rather than centralising in a single `providers.dart`.
