---
name: "flutter-riverpod"
description: "Riverpod (v3.3.x) compile-safe state management for Flutter with context-free reactive programming and code generation. Use this skill when implementing Riverpod providers with @riverpod annotations, handling AsyncValue for loading/error/data states, using ref.watch/ref.read/ref.listen for provider access, setting up ProviderScope as app root, implementing StateNotifier/StateNotifierProvider, using FutureProvider/StreamProvider for async data, enabling auto-retry and auto-dispose mechanisms, utilizing family/autoDispose modifiers, implementing provider dependencies and combining providers, migrating from Provider to Riverpod, debugging ProviderNotFoundException errors, testing providers in isolation, preventing memory leaks with autoDispose and ref.onDispose, avoiding UnmountedRefException in async operations, implementing lifecycle-aware state updates with ref.mounted checks, or choosing between ref.read and ref.watch for optimal rebuild performance. Ideal for apps requiring compile-time safety, testable state without BuildContext dependency, global state access from anywhere, complex async data flows, memory leak prevention, or avoiding ChangeNotifier performance pitfalls."
metadata:
  last_modified: "2026-03-31 10:45:00 (GMT+8)"
---

# Riverpod State Management Guide (v3.3.x)

## Goal
Implement state management using Riverpod, the spiritual successor to Provider with compile-time safety and context-free global access. Riverpod 3.x introduces automatic retry mechanisms, provider auto-pause, and enhanced code generation patterns. This guide covers both code-generation (recommended) and manual approaches.

## Process

### Phase 1: Install Dependencies

**For Code Generation (Recommended)**:
```yaml
dependencies:
  flutter_riverpod: ^3.3.1
  riverpod_annotation: ^3.3.1

dev_dependencies:
  build_runner: ^2.4.0
  riverpod_generator: ^3.3.1
```

**For Manual Approach**:
```yaml
dependencies:
  flutter_riverpod: ^3.3.1
```

### Phase 2: Setup ProviderScope

Wrap the app root with `ProviderScope`:

```dart
void main() {
  runApp(
    ProviderScope(
      child: MyApp(),
    ),
  );
}
```

### Phase 3: Create Providers

#### A. Code Generation Approach (Recommended)

**Simple Provider**:
```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'providers.g.dart';

@riverpod
String greeting(Ref ref) {
  return 'Hello, Riverpod!';
}

// Generated: greetingProvider
```

**Async Provider with Auto-Retry**:
```dart
@Riverpod(retry: myRetryStrategy)
Future<User> userProfile(Ref ref, String userId) async {
  final response = await http.get('/users/$userId');
  return User.fromJson(response.data);
}

// Custom retry strategy (max 5 attempts with exponential backoff)
Duration? myRetryStrategy(int retryCount, Object error) {
  if (retryCount >= 5) return null; // Stop retrying
  return Duration(milliseconds: 200 * (1 << retryCount));
}

// Disable retry entirely
@Riverpod(retry: (_, __) => null)
Future<Data> noRetryData(Ref ref) async { ... }
```

**Stateful Provider (Notifier Pattern)**:
```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0; // Initial state

  void increment() => state++;
  void decrement() => state--;
  void reset() => state = 0;
}

// Generated: counterProvider
```

**Async Notifier** (for mutable async state):
```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    return await fetchTodos();
  }

  Future<void> addTodo(String title) async {
    state = AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final newTodo = await apiClient.createTodo(title);
      final current = await future;
      return [...current, newTodo];
    });
  }
}
```

#### B. Manual Approach

**StateProvider** (for simple mutable state):
```dart
final counterProvider = StateProvider<int>((ref) => 0);
```

**FutureProvider** (for async data):
```dart
final userProvider = FutureProvider.autoDispose.family<User, String>(
  (ref, userId) async {
    return await fetchUser(userId);
  },
);
```

**StateNotifierProvider** (for complex state):
```dart
class CounterNotifier extends StateNotifier<int> {
  CounterNotifier() : super(0);
  
  void increment() => state++;
  void decrement() => state--;
}

final counterProvider = StateNotifierProvider<CounterNotifier, int>(
  (ref) => CounterNotifier(),
);
```

### Phase 4: Consume Providers in Widgets

**Using ConsumerWidget**:
```dart
class HomeScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final greeting = ref.watch(greetingProvider);
    final counter = ref.watch(counterProvider);
    
    return Scaffold(
      body: Column(
        children: [
          Text(greeting),
          Text('Count: $counter'),
          ElevatedButton(
            onPressed: () => ref.read(counterProvider.notifier).increment(),
            child: Text('Increment'),
          ),
        ],
      ),
    );
  }
}
```

**Using Consumer widget**:
```dart
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer(
      builder: (context, ref, child) {
        final count = ref.watch(counterProvider);
        return Text('Count: $count');
      },
    );
  }
}
```

**Using Hooks (with flutter_hooks)**:
```dart
class MyWidget extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    final textController = useTextEditingController();
    
    return TextField(controller: textController);
  }
}
```

### Phase 5: Handle Async States with AsyncValue

```dart
final userAsyncProvider = FutureProvider<User>((ref) => fetchUser());

// In widget
Widget build(BuildContext context, WidgetRef ref) {
  final userAsync = ref.watch(userAsyncProvider);
  
  return userAsync.when(
    data: (user) => Text('Hello, ${user.name}'),
    loading: () => CircularProgressIndicator(),
    error: (error, stack) => Text('Error: $error'),
  );
}

// Alternative pattern matching
final userAsync = ref.watch(userAsyncProvider);
if (userAsync.isLoading) return CircularProgressIndicator();
if (userAsync.hasError) return Text('Error: ${userAsync.error}');
if (userAsync.hasValue) return Text('User: ${userAsync.value!.name}');
```

### Phase 6: Advanced Patterns

#### Provider Dependencies

```dart
@riverpod
Future<User> user(Ref ref, String userId) async {
  return await fetchUser(userId);
}

@riverpod
Future<List<Post>> userPosts(Ref ref, String userId) async {
  // Depend on another provider
  final user = await ref.watch(userProvider(userId).future);
  return await fetchPostsForUser(user.id);
}
```

#### Auto-Dispose & Family Modifiers

```dart
// Auto-dispose when no longer watched
@riverpod
Future<Data> data(Ref ref) async { ... }

// Family for parameterized providers
@riverpod
Future<User> user(Ref ref, String userId) async { ... }

// Combined
final userProvider = FutureProvider.autoDispose.family<User, String>(
  (ref, userId) => fetchUser(userId),
);
```

#### Preventing Auto-Pause

If a provider needs to continue listening even when off-screen:

```dart
TickerMode(
  enabled: true, // Force provider to stay active
  child: Consumer(...),
)
```

---

## ⚠️ Critical Riverpod 3.x Pitfalls

### UnmountedRefException Prevention

**Problem:** Using `ref` after provider disposal causes runtime crashes with `StateError` or `UnmountedRefException`.

**Root Cause:** Async operations complete after the widget/provider has been disposed, attempting to update state on a dead reference.

**✅ Solutions:**

1. **Always check `ref.mounted` after async operations:**
```dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  bool build() => false;

  Future<void> login(String email, String password) async {
    await apiClient.login(email, password); // Network delay
    
    // ✅ CRITICAL: Check if ref is still alive before state update
    if (!ref.mounted) return;
    
    state = true; // Safe to update state
  }
}
```

2. **Use `ref.onDispose` to cancel pending work:**
```dart
@riverpod
Future<Data> pollingData(Ref ref) async {
  final timer = Timer.periodic(Duration(seconds: 5), (_) {
    // Poll server
  });
  
  // ✅ Cancel timer when provider disposes
  ref.onDispose(() {
    timer.cancel();
  });
  
  return fetchInitialData();
}
```

3. **Cancel HTTP requests and streams:**
```dart
@riverpod
Future<User> userData(Ref ref, String userId) async {
  final cancelToken = CancelToken();
  
  // ✅ Cancel pending request on disposal
  ref.onDispose(() {
    cancelToken.cancel();
  });
  
  final response = await dio.get('/users/$userId', cancelToken: cancelToken);
  return User.fromJson(response.data);
}
```

**❌ Anti-pattern:**
```dart
// ❌ DANGER: No ref.mounted check after await
Future<void> badLogin() async {
  await apiClient.login(); // Widget might dispose during this
  state = true; // 💥 Crash if widget already disposed!
}
```

---

### Memory Leak Prevention Checklist

**Must-do checklist for every provider:**

- ✅ **Use `autoDispose` for short-lived providers** (default in code generation)
- ✅ **Clean up resources in `ref.onDispose`** (timers, streams, listeners, HTTP requests)
- ✅ **Check `ref.mounted` before state updates in async methods**
- ✅ **Use `AsyncValue.guard` for async operations** (automatically catches errors)
- ✅ **Cancel streams with `ref.listen`** when no longer needed

**Example - Comprehensive Cleanup:**
```dart
@riverpod
class LocationTracker extends _$LocationTracker {
  StreamSubscription? _subscription;
  Timer? _heartbeatTimer;
  
  @override
  Stream<Location> build() {
    // ✅ Setup cleanup for all resources
    ref.onDispose(() {
      _subscription?.cancel();
      _heartbeatTimer?.cancel();
      print('LocationTracker disposed - all resources cleaned');
    });
    
    // Start heartbeat
    _heartbeatTimer = Timer.periodic(Duration(seconds: 30), (_) {
      sendHeartbeat();
    });
    
    return locationService.stream;
  }
  
  void sendHeartbeat() {
    // ✅ Check mounted before operations
    if (!ref.mounted) return;
    
    apiClient.ping();
  }
}
```

**When to avoid `autoDispose`:**
- Global singletons (Dio client, database instance)
- User authentication state
- App-wide configuration

**❌ Anti-patterns:**
```dart
// ❌ No cleanup - timer keeps running after disposal
@riverpod
class BadTimer extends _$BadTimer {
  @override
  int build() {
    Timer.periodic(Duration(seconds: 1), (_) {
      state++; // Memory leak! Timer never cancelled
    });
    return 0;
  }
}

// ❌ Stream subscription leak
@riverpod
Future<List<Message>> badMessages(Ref ref) async {
  messageStream.listen((msg) {
    // Listener never cancelled - memory leak!
  });
  return [];
}
```

---

### ref.read vs ref.watch - Decision Guide

**Golden Rule:**
- **`ref.watch`**: Rebuilds when dependency changes (reactive)
- **`ref.read`**: One-time read, no rebuild (imperative)

**Usage Matrix:**

| Context | Use `ref.watch` | Use `ref.read` |
|---------|----------------|----------------|
| Widget `build()` method | ✅ Yes | ❌ No |
| Provider `build()` method | ✅ Yes | ❌ No |
| Button callbacks (`onPressed`) | ❌ No | ✅ Yes |
| `initState`, `dispose` | ❌ No | ✅ Yes |
| Inside Notifier methods | ⚠️ Rarely | ✅ Usually |
| Event handlers | ❌ No | ✅ Yes |

**✅ Best Practices:**

```dart
class HomeScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ✅ Use ref.watch in build() - rebuilds when counter changes
    final count = ref.watch(counterProvider);
    
    return Scaffold(
      body: Text('Count: $count'),
      floatingActionButton: FloatingActionButton(
        // ✅ Use ref.read in callbacks - no rebuild needed
        onPressed: () => ref.read(counterProvider.notifier).increment(),
        child: Icon(Icons.add),
      ),
    );
  }
}
```

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    // ✅ Use ref.watch to depend on authProvider
    final userId = ref.watch(authProvider).userId;
    return fetchTodosForUser(userId);
  }
  
  Future<void> addTodo(String title) async {
    // ✅ Use ref.read in methods - avoid unnecessary rebuilds
    final auth = ref.read(authProvider);
    
    state = await AsyncValue.guard(() async {
      await apiClient.createTodo(title, userId: auth.userId);
      return fetchTodosForUser(auth.userId);
    });
  }
}
```

**❌ Anti-patterns:**

```dart
// ❌ Using ref.watch in callback causes rebuild on every provider change
FloatingActionButton(
  onPressed: () {
    final notifier = ref.watch(counterProvider.notifier); // ❌ Wrong!
    notifier.increment();
  },
)

// ✅ Correct version
FloatingActionButton(
  onPressed: () {
    ref.read(counterProvider.notifier).increment(); // ✅ Correct!
  },
)
```

```dart
// ❌ Using ref.watch in Notifier method causes cascade rebuilds
@riverpod
class BadNotifier extends _$BadNotifier {
  void doSomething() {
    final auth = ref.watch(authProvider); // ❌ Causes rebuild loop!
    // ...
  }
}

// ✅ Correct version
@riverpod
class GoodNotifier extends _$GoodNotifier {
  void doSomething() {
    final auth = ref.read(authProvider); // ✅ One-time read
    // ...
  }
}
```

---

### AsyncValue Best Practices

**Always use exhaustive `.when()` handling:**

```dart
// ✅ Exhaustive handling - covers all states
Widget build(BuildContext context, WidgetRef ref) {
  final userAsync = ref.watch(userProvider);
  
  return userAsync.when(
    data: (user) => UserProfile(user),
    loading: () => CircularProgressIndicator(),
    error: (error, stack) => ErrorWidget(error, stack),
  );
}
```

**Use `.whenData()` for data-only transformations:**

```dart
// ✅ Transform data while preserving loading/error states
final userNameAsync = ref.watch(userProvider).whenData(
  (user) => user.name,
);
```

**Use `AsyncValue.guard()` for error handling:**

```dart
// ✅ Automatically catches errors and wraps in AsyncValue.error
Future<void> updateProfile(User user) async {
  state = const AsyncValue.loading();
  
  state = await AsyncValue.guard(() async {
    await apiClient.updateUser(user);
    return user; // Return new state on success
  });
  
  // If apiClient throws, state becomes AsyncValue.error automatically
}
```

**Pattern matching with `switch` (Dart 3+):**

```dart
// ✅ Type-safe pattern matching
final widget = switch (userAsync) {
  AsyncData(:final value) => Text('Hello, ${value.name}'),
  AsyncLoading() => CircularProgressIndicator(),
  AsyncError(:final error) => Text('Error: $error'),
  _ => SizedBox.shrink(),
};
```

**❌ Anti-patterns:**

```dart
// ❌ Non-exhaustive handling - crashes if state is error
final user = userAsync.value!; // 💥 Throws if loading or error

// ❌ Manual try-catch instead of AsyncValue.guard
try {
  final result = await apiClient.fetch();
  state = AsyncValue.data(result);
} catch (e, st) {
  state = AsyncValue.error(e, st);
}

// ✅ Use guard instead
state = await AsyncValue.guard(() => apiClient.fetch());
```

**Handling partial updates (optimistic UI):**

```dart
// ✅ Preserve old data during loading
Future<void> refresh() async {
  // Keep old data visible while loading
  state = const AsyncValue.loading().copyWithPrevious(state);
  
  state = await AsyncValue.guard(() => fetchNewData());
}
```

---

## Reference Documentation

- [Riverpod Best Practices (v3.x)](./references/riverpod.md) - Complete guide with code generation patterns
- [State Management Overview](./references/state-management-overview.md) - Core concepts and decision logic

---

## Constraints

* **No Context Required**: Providers are globally accessible via `ref`, eliminating context-related errors.
* **Compile-Time Safety**: Use code generation (`@riverpod`) to catch errors at compile time instead of runtime.
* **Immutable State**: StateNotifier state must be immutable. Use Freezed for complex state classes.
* **Auto-Dispose by Default**: In code generation, providers auto-dispose unless marked `keepAlive: true`.
* **Ref Lifecycle**: Never store `ref` in variables or pass it outside the provider scope.
* **AsyncValue Guards**: Always use `AsyncValue.guard()` when updating async state to handle errors gracefully.
* **⚠️ CRITICAL - Mounted Check**: Always check `ref.mounted` after `await` calls before updating state to prevent UnmountedRefException.
* **⚠️ CRITICAL - Resource Cleanup**: Use `ref.onDispose` to cancel timers, streams, subscriptions, and HTTP requests to prevent memory leaks.
* **⚠️ CRITICAL - ref.read in Callbacks**: Never use `ref.watch` in event handlers or Notifier methods - use `ref.read` to avoid cascade rebuilds.
