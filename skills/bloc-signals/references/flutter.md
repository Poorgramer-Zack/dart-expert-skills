# BlocSignal in Flutter

Verified against `bloc_signals_flutter` 1.5.0 source.

## Provide

```dart
import 'package:bloc_signals_flutter/bloc_signals_flutter.dart';

BlocSignalProvider(
  create: (context) => CounterBloc(), // lazy by default; BlocSignalProvider owns and closes it
  child: const CounterScreen(),
);

BlocSignalProvider(
  create: (_) => CounterBloc(),
  lazy: false, // create eagerly
  child: const CounterScreen(),
);

BlocSignalProvider.value(
  value: existingBloc, // does not own or close the bloc
  child: const CounterScreen(),
);
```

- Multiple providers: `MultiBlocSignalProvider`.
- `BlocSignalProvider` and `BlocSignalListener` take an optional `child`, so `MultiBlocSignalProvider` and `MultiBlocSignalListener` do not need placeholder children.

## Read the bloc

```dart
context.read<CounterBloc>().add(Increment()); // in callbacks
final bloc = context.read<CounterBloc>();
```

- `context.read<T>()` returns the instance without subscribing.
- `context.watch<T>()` rebuilds when the *provider instance* changes, not when state changes. Do not use it to react to state.

## Rebuild on state

```dart
BlocSignalBuilder<CounterBloc, int>(
  builder: (context, count) => Text('$count'),
);
```

- `BlocSignalBuilder` takes the bloc from the nearest provider. Pass `bloc:` to use an explicit instance.
- Inside `build`, `context.value<CounterBloc, int>()` reads the current value and subscribes to it.
- `context.state<CounterBloc, int>()` returns a `ReadonlySignal<int>` for composing with `computed()`.

## Listen for side effects

```dart
BlocSignalListener<CounterBloc, int>(
  listenWhen: (previous, current) => current > 10,
  listener: (context, count) => ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text('Reached $count')),
  ),
  child: const CounterBody(),
);
```

- `BlocSignalListener` does not run its listener on the initial value.
- `listenWhen(previous, current)` filters transitions. The listener receives only the current state.

## Consumer and selector

- `BlocSignalConsumer` combines `builder` and `listener`.
- `BlocSignalSelector<CounterBloc, int, V>` rebuilds only when the selected value changes.
- `context.select<CounterBloc, bool>((bloc) => bloc.stateValue > 0)` reads a derived value. It takes two type arguments (bloc type and result type), and the callback receives the bloc instance.

## flutter_hooks

Use the `signals_hooks` integration in a `HookWidget`:

```dart
class CounterHookView extends HookWidget {
  const CounterHookView({super.key});

  @override
  Widget build(BuildContext context) {
    final bloc = useMemoized(() => CounterBloc());
    final count = useSignalValue(bloc.state);
    return Text('$count');
  }
}
```

Dispose the bloc yourself when you create it inside a hook (use `useEffect` to call `bloc.close()` on dispose).

## Ownership rules

- A bloc created by `BlocSignalProvider(create:)` is closed automatically.
- A bloc created in a `StatefulWidget` must be created once in `initState` and closed in `dispose`. Creating it in `build` leaks it on every rebuild.
