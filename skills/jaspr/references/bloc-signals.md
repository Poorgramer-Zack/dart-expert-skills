# BlocSignal in Jaspr (`bloc_signals_jaspr`)

Verified against `bloc_signals_jaspr` 1.3.0, `bloc_signals` 1.7.0, and `jaspr` 0.23.5 with `dart analyze`.

## Setup

```yaml
dependencies:
  jaspr: ^0.23.5
  jaspr_router: ^0.9.0        # if you use routing
  bloc_signals: ^1.7.0
  bloc_signals_jaspr: ^1.3.0
```

`bloc_signals_jaspr` re-exports the core `bloc_signals` types. Import only `package:bloc_signals_jaspr/bloc_signals_jaspr.dart` plus `package:jaspr/dom.dart` for DOM elements.

## Define a bloc

```dart
import 'package:bloc_signals_jaspr/bloc_signals_jaspr.dart';

sealed class CounterEvent {}

final class Increment extends CounterEvent {}

class CounterBloc extends BlocSignal<CounterEvent, int> {
  CounterBloc() : super(initialState: 0) {
    on<Increment>((event, emit) => emit(stateValue + 1));
  }
}
```

Use `CubitSignal<int>` with a method (`void increment() => emit(stateValue + 1);`) when events are not needed.

## Provide the bloc

```dart
BlocSignalProvider(
  create: (_) => CounterBloc(), // owned and closed by the provider
  child: const CounterView(),
);

BlocSignalProvider.value(
  value: existingBloc, // not closed by the provider
  child: const CounterView(),
);
```

- `BlocSignalProvider` is a `Component`. It is not a Flutter `Widget`.
- Several providers: `MultiBlocSignalProvider`.

## Read, rebuild, and listen

```dart
class CounterView extends StatelessComponent {
  const CounterView({super.key});

  @override
  Component build(BuildContext context) {
    final bloc = context.read<CounterBloc>(); // in event callbacks
    return div([
      BlocSignalBuilder<CounterBloc, int>(
        builder: (context, count) => p([Component.text('Count: $count')]),
      ),
      button(onClick: () => bloc.add(Increment()), [Component.text('+1')]),
    ]);
  }
}
```

- `BlocSignalBuilder` rebuilds on state change. Pass `bloc:` to use an explicit instance, or let it read the nearest provider.
- `BlocSignalListener` runs side effects on state change and skips the initial value. Use `listenWhen` to filter. It also accepts `bloc:` and `child:`.
- `BlocSignalSelector<T, S, V>` rebuilds only when the selected value changes.
- `context.watch` tracks provider replacement, not state. Use `BlocSignalBuilder` for state.
- `context.select<T, R>((bloc) => ...)`, `context.value<T, S>()`, and `context.state<T, S>()` are available for derived values.

## Text components

Jaspr 0.23 deprecates the top-level `text()` function. Use `Component.text('...')` inside child lists.

## Server-side rendering and ownership

- A bloc created in `create:` is created during the render of that component. Keep the provider near the page that needs it. Do not create a global bloc at import time, because SSR runs for every request.
- Global state shared across requests leaks data between users. For SSR, put per-request blocs inside a provider in the page component, not in a top-level variable.
- Use a `BlocSignalProvider` on the client side for interactive islands. Keep data loading (`AsyncStatelessComponent` or `PreloadStateMixin`) on the server and pass the result into the bloc's initial state.

## Testing

Use `jaspr_test` for component tests and `blocSignalTest` (from `bloc_signals_test`) for bloc logic. Keep bloc logic tests free of Jaspr so they run as plain Dart tests.
