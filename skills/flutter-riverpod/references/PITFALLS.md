# Riverpod 3.x Common Pitfalls

## Table of Contents

- [UnmountedRefException Prevention](#unmountedrefexception-prevention)
- [Memory Leak Prevention](#memory-leak-prevention)
- [ref.read vs ref.watch](#refread-vs-refwatch)
- [AsyncValue Best Practices](#asyncvalue-best-practices)

---

## UnmountedRefException Prevention

**Problem:** Using `ref` after provider disposal causes `StateError` or `UnmountedRefException`.

**Root Cause:** Async operations complete after the widget/provider has been disposed, attempting to update state on a dead reference.

### ✅ Always check `ref.mounted` after async operations

```dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  bool build() => false;

  Future<void> login(String email, String password) async {
    await apiClient.login(email, password);

    if (!ref.mounted) return; // Guard before any state update
    state = true;
  }
}
```

### ✅ Cancel pending work with `ref.onDispose`

```dart
@riverpod
Future<Data> pollingData(Ref ref) async {
  final timer = Timer.periodic(const Duration(seconds: 5), (_) {
    // poll server
  });
  ref.onDispose(timer.cancel);
  return fetchInitialData();
}
```

### ✅ Cancel HTTP requests on disposal

```dart
@riverpod
Future<User> userData(Ref ref, String userId) async {
  final cancelToken = CancelToken();
  ref.onDispose(cancelToken.cancel);
  final response = await dio.get('/users/$userId', cancelToken: cancelToken);
  return User.fromJson(response.data);
}
```

### ❌ Anti-pattern

```dart
Future<void> badLogin() async {
  await apiClient.login(); // Widget may dispose during this await
  state = true; // 💥 Crashes if provider already disposed
}
```

---

## Memory Leak Prevention

**Checklist for every provider:**

- ✅ Use `autoDispose` for short-lived providers (default with code generation)
- ✅ Clean up all resources in `ref.onDispose` (timers, streams, listeners, HTTP requests)
- ✅ Check `ref.mounted` before state updates in async methods
- ✅ Use `AsyncValue.guard()` for async operations
- ✅ Cancel `.listen` subscriptions when no longer needed

**When to avoid `autoDispose`:**
- Global singletons (HTTP client, database instance)
- User authentication state
- App-wide configuration

### ✅ Comprehensive cleanup example

```dart
@riverpod
class LocationTracker extends _$LocationTracker {
  StreamSubscription? _subscription;
  Timer? _heartbeatTimer;

  @override
  Stream<Location> build() {
    ref.onDispose(() {
      _subscription?.cancel();
      _heartbeatTimer?.cancel();
    });

    _heartbeatTimer = Timer.periodic(const Duration(seconds: 30), (_) {
      if (ref.mounted) apiClient.ping();
    });

    return locationService.stream;
  }
}
```

### ❌ Anti-patterns

```dart
// Timer never cancelled — memory leak
@riverpod
class BadTimer extends _$BadTimer {
  @override
  int build() {
    Timer.periodic(const Duration(seconds: 1), (_) => state++);
    return 0;
  }
}

// Stream subscription never cancelled
@riverpod
Future<List<Message>> badMessages(Ref ref) async {
  messageStream.listen((msg) {}); // Listener leaks
  return [];
}
```

---

## ref.read vs ref.watch

**Golden Rule:**
- `ref.watch` — reactive, rebuilds the caller when the dependency changes
- `ref.read` — one-time snapshot, no rebuild

| Context | ref.watch | ref.read |
|---------|-----------|----------|
| Widget `build()` | ✅ | ❌ |
| Provider `build()` | ✅ | ❌ |
| Button `onPressed` / event handlers | ❌ | ✅ |
| `initState`, `dispose` | ❌ | ✅ |
| Inside Notifier methods | ⚠️ Rarely | ✅ Usually |

### ✅ Correct pattern

```dart
class HomeScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider); // reactive

    return Scaffold(
      body: Text('Count: $count'),
      floatingActionButton: FloatingActionButton(
        onPressed: () => ref.read(counterProvider.notifier).increment(), // one-shot
        child: const Icon(Icons.add),
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
    final userId = ref.watch(authProvider).userId; // reactive dependency
    return fetchTodosForUser(userId);
  }

  Future<void> addTodo(String title) async {
    final auth = ref.read(authProvider); // one-shot in method
    state = await AsyncValue.guard(() async {
      await apiClient.createTodo(title, userId: auth.userId);
      return fetchTodosForUser(auth.userId);
    });
  }
}
```

### ❌ Anti-patterns

```dart
// ref.watch in callback triggers rebuild on every provider change
FloatingActionButton(
  onPressed: () => ref.watch(counterProvider.notifier).increment(), // ❌
)

// ref.watch in Notifier method causes cascade rebuild loop
class BadNotifier extends _$BadNotifier {
  void doSomething() {
    final auth = ref.watch(authProvider); // ❌ rebuild loop
  }
}
```

---

## AsyncValue Best Practices

### ✅ Exhaustive `.when()` handling

```dart
return userAsync.when(
  data: (user) => UserProfile(user),
  loading: () => const CircularProgressIndicator(),
  error: (error, stack) => ErrorWidget(error, stack),
);
```

### ✅ `.whenData()` for data-only transformations

```dart
// Preserves loading/error states while transforming data
final userNameAsync = ref.watch(userProvider).whenData((u) => u.name);
```

### ✅ `AsyncValue.guard()` for error handling

```dart
Future<void> updateProfile(User user) async {
  state = const AsyncValue.loading();
  state = await AsyncValue.guard(() async {
    await apiClient.updateUser(user);
    return user;
  });
  // On throw, state automatically becomes AsyncValue.error
}
```

### ✅ Dart 3 pattern matching

```dart
final widget = switch (userAsync) {
  AsyncData(:final value) => Text('Hello, ${value.name}'),
  AsyncLoading() => const CircularProgressIndicator(),
  AsyncError(:final error) => Text('Error: $error'),
  _ => const SizedBox.shrink(),
};
```

### ✅ Optimistic UI with `copyWithPrevious`

```dart
Future<void> refresh() async {
  // Keep stale data visible during reload
  state = const AsyncValue.loading().copyWithPrevious(state);
  state = await AsyncValue.guard(fetchNewData);
}
```

### ❌ Anti-patterns

```dart
final user = userAsync.value!; // 💥 throws if loading or error

// Manual try-catch is verbose and error-prone
try {
  state = AsyncValue.data(await apiClient.fetch());
} catch (e, st) {
  state = AsyncValue.error(e, st);
}
// ✅ Replace with: state = await AsyncValue.guard(() => apiClient.fetch());
```
