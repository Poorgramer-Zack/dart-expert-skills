---
name: "flutter-bloc"
description: "BLoC (v9.x) event-driven state management for Flutter with unidirectional data flow architecture using flutter_bloc package. Use this skill when implementing BLoC pattern with bloc/cubit classes, handling Event-State mapping, using BlocBuilder/BlocListener/BlocConsumer widgets, implementing event transformers (concurrent/sequential/restartable/droppable) for concurrency control, setting up MultiBlocProvider or BlocProvider dependency injection, integrating with Freezed for immutable sealed union types and exhaustive pattern matching, testing blocs with blocTest, implementing complex state machines with multiple states, handling error states and loading indicators with centralized error handling, avoiding common errors like emitting after close or mutable state mutations, or requiring strict event sourcing and audit trails. Ideal for enterprise apps, complex business logic separation, apps requiring precise concurrency control (search debouncing, form submission deduplication), or apps requiring event replay debugging and comprehensive state testing."
metadata:
  last_modified: "2026-03-31 10:45:00 (GMT+8)"
---

# BLoC State Management Guide (v9.x)

## Goal
Implement BLoC (Business Logic Component) pattern for strict separation between UI and business logic using event-driven unidirectional data flow. BLoC v9.x provides both full Bloc (with Events) and lightweight Cubit (without Events) implementations.

## Process

### Phase 1: Install Dependencies

```yaml
dependencies:
  flutter_bloc: ^9.1.1
  equatable: ^2.0.5  # For value equality

dev_dependencies:
  bloc_test: ^9.1.0  # For testing
```

### Phase 2: Choose Implementation Pattern

- **Cubit**: Lightweight, no events. Use for simple state changes (e.g., toggle dark mode, counter).
- **Bloc**: Full event-driven. Use when you need event tracking, transformers, or complex flows (e.g., login, search with debounce).

### Phase 3A: Implement Cubit (Lightweight)

**Define State**:
```dart
import 'package:equatable/equatable.dart';

class CounterState extends Equatable {
  final int value;
  
  const CounterState({required this.value});
  
  @override
  List<Object?> get props => [value];
}
```

**Create Cubit**:
```dart
import 'package:flutter_bloc/flutter_bloc.dart';

class CounterCubit extends Cubit<CounterState> {
  CounterCubit() : super(const CounterState(value: 0));
  
  void increment() => emit(state.copyWith(value: state.value + 1));
  void decrement() => emit(state.copyWith(value: state.value - 1));
  void reset() => emit(const CounterState(value: 0));
}
```

### Phase 3B: Implement Bloc (Full Event-Driven)

**Define Events**:
```dart
abstract class CounterEvent extends Equatable {
  const CounterEvent();
  
  @override
  List<Object?> get props => [];
}

class CounterIncremented extends CounterEvent {}
class CounterDecremented extends CounterEvent {}
class CounterReset extends CounterEvent {}
```

**Define State**:
```dart
class CounterState extends Equatable {
  final int value;
  
  const CounterState({required this.value});
  
  @override
  List<Object?> get props => [value];
}
```

**Create Bloc**:
```dart
class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(const CounterState(value: 0)) {
    on<CounterIncremented>(_onIncremented);
    on<CounterDecremented>(_onDecremented);
    on<CounterReset>(_onReset);
  }
  
  void _onIncremented(CounterIncremented event, Emitter<CounterState> emit) {
    emit(CounterState(value: state.value + 1));
  }
  
  void _onDecremented(CounterDecremented event, Emitter<CounterState> emit) {
    emit(CounterState(value: state.value - 1));
  }
  
  void _onReset(CounterReset event, Emitter<CounterState> emit) {
    emit(const CounterState(value: 0));
  }
}
```

### Phase 4: Provide Bloc/Cubit to Widget Tree

```dart
void main() {
  runApp(
    MultiBlocProvider(
      providers: [
        BlocProvider(create: (context) => CounterCubit()),
        BlocProvider(create: (context) => AuthBloc()),
      ],
      child: MyApp(),
    ),
  );
}

// Or provide locally for a specific screen
class CounterScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (context) => CounterCubit(),
      child: CounterView(),
    );
  }
}
```

### Phase 5: Consume State in Widgets

**BlocBuilder** (rebuild on state changes):
```dart
BlocBuilder<CounterCubit, CounterState>(
  builder: (context, state) {
    return Text('Count: ${state.value}');
  },
)
```

**BlocListener** (side effects, no rebuild):
```dart
BlocListener<AuthBloc, AuthState>(
  listener: (context, state) {
    if (state is AuthSuccess) {
      Navigator.pushReplacementNamed(context, '/home');
    } else if (state is AuthFailure) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(state.error)),
      );
    }
  },
  child: LoginForm(),
)
```

**BlocConsumer** (combines Builder + Listener):
```dart
BlocConsumer<AuthBloc, AuthState>(
  listener: (context, state) {
    if (state is AuthFailure) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(state.error)),
      );
    }
  },
  builder: (context, state) {
    if (state is AuthLoading) {
      return CircularProgressIndicator();
    }
    return LoginButton();
  },
)
```

**Trigger Events**:
```dart
ElevatedButton(
  onPressed: () => context.read<CounterBloc>().add(CounterIncremented()),
  child: Text('Increment'),
)

// Or with Cubit (direct method call)
ElevatedButton(
  onPressed: () => context.read<CounterCubit>().increment(),
  child: Text('Increment'),
)
```

### Phase 6: BLoC 9.x Concurrency & Event Transformers

#### Event Transformer Decision Guide

BLoC provides four built-in transformers to control event concurrency. Choose based on your use case:

| Transformer | When to Use | Behavior | Example Use Case |
|-------------|-------------|----------|------------------|
| **concurrent** (default) | Independent parallel events | Process all events simultaneously | Multiple independent API calls, analytics events |
| **sequential** | Order-dependent operations | Queue events, process one at a time | Multi-step form wizard, ordered database transactions |
| **restartable** | Only latest event matters | Cancel previous incomplete event, start new one | Search bar typing, autocomplete, real-time filters |
| **droppable** | Prevent duplicate processing | Ignore new events while processing current one | Submit button spam prevention, one-time API calls |

**Setup** (add dependency):
```yaml
dependencies:
  bloc_concurrency: ^0.3.0
```

**Implementation Examples**:

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';

class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc() : super(SearchInitial()) {
    // ✅ RESTARTABLE: Cancel previous search when user types new character
    on<SearchQueryChanged>(
      _onQueryChanged,
      transformer: restartable(),
    );
    
    // ✅ DROPPABLE: Prevent double-submit when user clicks button twice
    on<FormSubmitted>(
      _onFormSubmitted,
      transformer: droppable(),
    );
    
    // ✅ SEQUENTIAL: Ensure operations complete in order
    on<DataSyncRequested>(
      _onDataSync,
      transformer: sequential(),
    );
    
    // ✅ CONCURRENT (default): Run multiple independent tasks simultaneously
    on<AnalyticsEventTracked>(
      _onAnalytics,
      // No transformer needed - concurrent is default
    );
  }
  
  Future<void> _onQueryChanged(
    SearchQueryChanged event, 
    Emitter<SearchState> emit,
  ) async {
    emit(SearchLoading());
    // Previous search API call is automatically cancelled if user types again
    final results = await searchRepository.search(event.query);
    emit(SearchSuccess(results));
  }
  
  Future<void> _onFormSubmitted(
    FormSubmitted event,
    Emitter<SearchState> emit,
  ) async {
    // If this is running, subsequent FormSubmitted events are ignored
    emit(FormSubmitting());
    await api.submitForm(event.data);
    emit(FormSubmitted());
  }
}
```

#### Custom Debounce Transformer

For search with delay (wait for user to stop typing):

```dart
import 'package:stream_transform/stream_transform.dart';

EventTransformer<E> debounce<E>(Duration duration) {
  return (events, mapper) => events.debounce(duration).switchMap(mapper);
}

// Usage
on<SearchQueryChanged>(
  _onQueryChanged,
  transformer: debounce(const Duration(milliseconds: 300)),
);
```

### Phase 7: Common BLoC Errors & Solutions

#### Error 1: Emitting After Bloc Close

**Problem**: 
```dart
// ❌ BAD: Can cause "Cannot emit new states after calling close"
class MyBloc extends Bloc<MyEvent, MyState> {
  Future<void> _onEvent(MyEvent event, Emitter<MyState> emit) async {
    await Future.delayed(Duration(seconds: 5));
    emit(NewState()); // Bloc might be closed by now!
  }
}
```

**Solution**:
```dart
// ✅ GOOD: Check isClosed before emitting
Future<void> _onEvent(MyEvent event, Emitter<MyState> emit) async {
  await Future.delayed(Duration(seconds: 5));
  if (!isClosed) {
    emit(NewState());
  }
}

// ✅ BETTER: Use emit.isDone for async operations
Future<void> _onEvent(MyEvent event, Emitter<MyState> emit) async {
  await Future.delayed(Duration(seconds: 5));
  if (!emit.isDone) {  // More reliable for emitter context
    emit(NewState());
  }
}
```

**Always close Blocs properly**:
```dart
// In StatefulWidget
@override
void dispose() {
  myBloc.close();  // Always close to prevent memory leaks
  super.dispose();
}
```

#### Error 2: Mutable State Causing UI Bugs

**Problem**:
```dart
// ❌ BAD: Mutating state in-place breaks reactivity
class UserState {
  final List<User> users; // Mutable list!
  UserState(this.users);
}

void _onUserAdded(UserAdded event, Emitter<UserState> emit) {
  state.users.add(event.user); // BlocBuilder won't rebuild!
  emit(state); // Same reference, no change detected
}
```

**Solution with Equatable**:
```dart
// ✅ GOOD: Create new instances with Equatable
class UserState extends Equatable {
  final List<User> users;
  
  const UserState(this.users);
  
  @override
  List<Object?> get props => [users];
}

void _onUserAdded(UserAdded event, Emitter<UserState> emit) {
  emit(UserState([...state.users, event.user])); // New list instance
}
```

**Solution with Freezed** (Best):
```dart
// ✅ BEST: Freezed guarantees immutability
@freezed
class UserState with _$UserState {
  const factory UserState({
    required List<User> users,
  }) = _UserState;
}

void _onUserAdded(UserAdded event, Emitter<UserState> emit) {
  emit(state.copyWith(
    users: [...state.users, event.user],
  ));
}
```

#### Error 3: Missing Centralized Error Handling

**Problem**:
```dart
// ❌ BAD: Scattered try-catch blocks in every event handler
void _onEvent1(Event1 event, Emitter<MyState> emit) async {
  try {
    // logic
  } catch (e) {
    emit(ErrorState(e.toString()));
  }
}

void _onEvent2(Event2 event, Emitter<MyState> emit) async {
  try {
    // logic
  } catch (e) {
    emit(ErrorState(e.toString())); // Duplicated error handling
  }
}
```

**Solution**:
```dart
// ✅ GOOD: Override onError for centralized handling
class MyBloc extends Bloc<MyEvent, MyState> {
  @override
  void onError(Object error, StackTrace stackTrace) {
    // Centralized logging
    logger.error('Bloc error', error, stackTrace);
    
    // Send to crash reporting
    crashReporter.report(error, stackTrace);
    
    super.onError(error, stackTrace);
  }
  
  // Event handlers can now focus on logic
  Future<void> _onEvent(MyEvent event, Emitter<MyState> emit) async {
    emit(LoadingState());
    final data = await repository.fetch(); // Errors caught by onError
    emit(SuccessState(data));
  }
}

// For user-facing error states, use addError
Future<void> _onLoginEvent(LoginEvent event, Emitter<AuthState> emit) async {
  try {
    await authRepository.login(event.username, event.password);
    emit(AuthSuccess());
  } on InvalidCredentialsException catch (e) {
    emit(AuthFailure(e.message)); // User-facing error
  } catch (e) {
    addError(e); // Triggers onError for unexpected errors
  }
}
```

### Phase 8: Freezed Integration for Immutable States

**Why Freezed with BLoC?**
- ✅ **Guaranteed Immutability**: No accidental state mutations
- ✅ **Exhaustive Pattern Matching**: Compiler forces you to handle all state cases with `when`/`map`
- ✅ **Less Boilerplate**: Auto-generates `copyWith`, `==`, `hashCode`, `toString`
- ✅ **Union Types**: Multiple state variants in one sealed class

**Setup**:
```yaml
dependencies:
  freezed_annotation: ^2.4.1

dev_dependencies:
  build_runner: ^2.4.6
  freezed: ^2.4.5
```

**Define States with Freezed**:
```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'auth_state.freezed.dart';

@freezed
class AuthState with _$AuthState {
  const factory AuthState.initial() = AuthInitial;
  const factory AuthState.loading() = AuthLoading;
  const factory AuthState.authenticated(User user) = AuthAuthenticated;
  const factory AuthState.unauthenticated() = AuthUnauthenticated;
  const factory AuthState.error(String message) = AuthError;
}
```

**Generate code**:
```bash
dart run build_runner build --delete-conflicting-outputs
```

**Exhaustive Pattern Matching in Widgets**:
```dart
BlocBuilder<AuthBloc, AuthState>(
  builder: (context, state) {
    // 🔥 when() forces you to handle ALL cases - compiler error if you miss one!
    return state.when(
      initial: () => Text('Please login'),
      loading: () => CircularProgressIndicator(),
      authenticated: (user) => Text('Welcome, ${user.name}'),
      unauthenticated: () => LoginForm(),
      error: (msg) => Text('Error: $msg'),
    );
  },
)

// 🔥 map() for more control (returns typed values)
final String title = state.map(
  initial: (_) => 'Login',
  loading: (_) => 'Logging in...',
  authenticated: (state) => 'Hello ${state.user.name}',
  unauthenticated: (_) => 'Login',
  error: (_) => 'Error',
);

// 🔥 maybeWhen() for partial handling with orElse fallback
state.maybeWhen(
  authenticated: (user) => showUserProfile(user),
  orElse: () => showLoginScreen(),
);
```

**Freezed Events with Data**:
```dart
@freezed
class AuthEvent with _$AuthEvent {
  const factory AuthEvent.loginRequested({
    required String username,
    required String password,
  }) = _LoginRequested;
  
  const factory AuthEvent.logoutRequested() = _LogoutRequested;
  
  const factory AuthEvent.tokenRefreshRequested({
    required String refreshToken,
  }) = _TokenRefreshRequested;
}

// In BLoC
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  AuthBloc() : super(const AuthState.initial()) {
    on<AuthEvent>((event, emit) async {
      await event.map(
        loginRequested: (e) => _handleLogin(e, emit),
        logoutRequested: (e) => _handleLogout(e, emit),
        tokenRefreshRequested: (e) => _handleRefresh(e, emit),
      );
    });
  }
  
  Future<void> _handleLogin(_LoginRequested event, Emitter emit) async {
    emit(const AuthState.loading());
    try {
      final user = await authRepo.login(event.username, event.password);
      emit(AuthState.authenticated(user));
    } catch (e) {
      emit(AuthState.error(e.toString()));
    }
  }
}
```

**Complex States with copyWith**:
```dart
@freezed
class UserState with _$UserState {
  const factory UserState({
    required List<User> users,
    required bool isLoading,
    String? errorMessage,
    User? selectedUser,
  }) = _UserState;
  
  // Factory for initial state
  factory UserState.initial() => const UserState(
    users: [],
    isLoading: false,
  );
}

// Usage in BLoC
void _onUserAdded(UserAdded event, Emitter<UserState> emit) {
  emit(state.copyWith(
    users: [...state.users, event.user],
    selectedUser: event.user,
  ));
}
```

---

## Reference Documentation

- [BLoC Best Practices (v9.x)](./references/bloc.md) - Complete guide with event transformers
- [State Management Overview](./references/state-management-overview.md) - Core concepts

---

## Constraints

* **Immutable States**: All states MUST be immutable. Use `Equatable` or `Freezed`.
* **No UI in Bloc**: Blocs/Cubits MUST NOT import Flutter widgets or BuildContext.
* **Single Responsibility**: Each Bloc should handle one feature domain.
* **Event Naming**: Use past tense (e.g., `UserLoggedIn`, not `UserLogin`).
* **State Naming**: Use nouns or adjectives (e.g., `AuthSuccess`, not `AuthSucceeded`).
* **Always Close**: Blocs/Cubits MUST be closed when disposed to prevent memory leaks.
