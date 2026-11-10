# BlocSignal core API

Verified against `bloc_signals` 1.7.0 source.

## Choose the container

| Type | Use when |
|---|---|
| `CubitSignal<State>` | Public methods change state (`increment()`). |
| `BlocSignal<Event, State>` | Events are dispatched and handled by `on<E>`. |
| `CubitSignalMixin` / `BlocSignalMixin` | The class already extends another base class (for example `ChangeNotifier`). Call `initCubitSignal(initialState: ...)` in the constructor. |

## Cubit

```dart
import 'package:bloc_signals/bloc_signals.dart';

class CounterCubit extends CubitSignal<int> {
  CounterCubit() : super(initialState: 0);

  void increment() => emit(stateValue + 1);
}
```

## Bloc with events

```dart
import 'package:bloc_signals/bloc_signals.dart';

sealed class CounterEvent {}

final class Increment extends CounterEvent {}

final class Decrement extends CounterEvent {}

class CounterBloc extends BlocSignal<CounterEvent, int> {
  CounterBloc() : super(initialState: 0) {
    on<Increment>((event, emit) => emit(stateValue + 1));
    on<Decrement>((event, emit) => emit(stateValue - 1));
  }
}

final bloc = CounterBloc();
bloc.add(Increment());
print(bloc.stateValue); // 1, updated synchronously
bloc.close();
```

- `on<E>` accepts an optional `transformer:` (`EventTransformer`) or `blocTransformer:` (`BlocEventTransformer`). Set only one of them.
- Registering the same exact event type twice throws `StateError`. Registration is runtime routing, so the compiler does not check sealed-class exhaustiveness.
- If you override `onEvent` or `onChange`, call `super`.

## Reading state

| Member | Type | Use |
|---|---|---|
| `stateValue` / `value` | `State` | Read the current value in methods and handlers. |
| `state` | `ReadonlySignal<State>` | Subscribe reactively (widgets, `computed`, `effect`). |

## Effects and lifecycle

```dart
class CounterCubit extends CubitSignal<int> {
  CounterCubit() : super(initialState: 0) {
    createEffect(() {
      if (stateValue > 10) print('double digits: $stateValue');
    });
  }

  void increment() => emit(stateValue + 1);
}
```

- `createEffect(callback, options:, onDispose:)` returns a disposer. `close()` runs all disposers.
- `close()` returns `Future<void>`. It is safe to call twice.
- After `close()`, new events are dropped. An `emit` after close asserts in debug mode and does nothing in release mode.
- Avoid `emit` during a `build` method, and never create an `effect` or `computed` inside `build`.

## Equality

- `emit` skips a value equal (`==`) to the current state.
- Override the equality check with the `equals` option when needed, for example to make every emit notify (`equals => false`) or to use identity.

## Event transformers

- Use `droppable` for taps, `restartable` for search-as-you-type, and `sequential` for ordered writes.
- Transformers are plain Dart higher-order functions. They do not use streams, so they add no stream subscriptions.

## Sources

- Upstream core guide: `plugins/bloc-signals/skills/bloc-signals/core.md` in the BlocSignal repository.
