# Migrating from flutter_bloc to BlocSignal

Read this before replacing `bloc`, `flutter_bloc`, or `bloc_test` code. The mapping below comes from the upstream migration guide and the `bloc_signals` 1.7.0 source.

## Code changes

| flutter_bloc | BlocSignal |
|---|---|
| `extends Bloc<E, S>` | `extends BlocSignal<E, S>` |
| `extends Cubit<S>` | `extends CubitSignal<S>` |
| `: super(0)` | `: super(initialState: 0)` |
| `state` (raw value) | `stateValue` or `value` |
| `emit` in stream-based handlers | `emit` is synchronous |
| `BlocProvider` | `BlocSignalProvider` |
| `MultiBlocProvider` | `MultiBlocSignalProvider` |
| `BlocBuilder` | `BlocSignalBuilder` |
| `BlocListener` | `BlocSignalListener` |
| `BlocConsumer` | `BlocSignalConsumer` |
| `BlocSelector` | `BlocSignalSelector` |
| `RepositoryProvider` | Not needed. Use `BlocSignalProvider` or your own DI. |
| `context.select<B, R>((b) => ...)` | Same idea. `context.select<B, R>` takes two type arguments, and the callback receives the bloc. |

Before:

```dart
class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0) {
    on<Increment>((event, emit) => emit(state + 1));
  }
}
```

After:

```dart
class CounterBloc extends BlocSignal<CounterEvent, int> {
  CounterBloc() : super(initialState: 0) {
    on<Increment>((event, emit) => emit(stateValue + 1));
  }
}
```

## Behavior differences that break code

- **No streams on the base classes.** `BlocSignalBase`, `CubitSignalMixin`, and `BlocSignalMixin` do not expose a `stream` getter. Code that calls `bloc.stream.listen(...)` must use `state` (a `ReadonlySignal`) or `createEffect`. To bring a `Stream` into BlocSignal, use `toBlocSignal`.
- **Synchronous updates.** After `add`, `stateValue` is already updated. Tests that waited for a microtask can drop the wait. Tests that relied on the delay need an explicit `wait:`.
- **Listener does not fire initially.** `BlocSignalListener` skips the first value. `BlocListener` in flutter_bloc behaves differently.
- **Equality skip.** An `emit` equal to the current state does nothing, so no listener and no transition runs.
- **`watch` does not rebuild on state.** Use `BlocSignalBuilder` or `context.value`.
- **One global observer.** `BlocSignalObserver` is a single slot. If you already have a logger, compose both observers.
- **Concurrency transformers** run as plain Dart functions. `bloc_concurrency` transformers are not used.

## Migrating a legacy bloc you do not want to rewrite

Use the bridge in the upstream migration guide. Wrap an existing `Stream` with `toBlocSignal`. Create the wrapper once, in `initState`, and close it in `dispose`. Creating it in `build` leaks subscriptions.

```dart
class _MyWidgetState extends State<MyWidget> {
  late final BlocSignalBase<int> _blocSignal;

  @override
  void initState() {
    super.initState();
    _blocSignal = widget.legacyBloc.stream.toBlocSignal(
      initialState: widget.legacyBloc.state,
    );
  }

  @override
  void dispose() {
    _blocSignal.close();
    super.dispose();
  }
}
```

The `Stream.toBlocSignal({initialState})` extension is defined in `bloc_signals` 1.7.0 (`lib/src/stream_adapter.dart`). Check the signature again if you use a different version.

## Checklist

1. Read the installed `bloc_signals` version.
2. Replace the base classes and constructors.
3. Replace `state` reads with `stateValue`.
4. Replace the widget classes.
5. Update tests: drop microtask waits, keep `blocSignalTest`.
6. Check for `stream` usages and listener logic that assumed the initial emission.
