# Testing BlocSignal

Verified against `bloc_signals_test` 1.0.3. The sample below passed with `flutter test`.

## Declarative tests with `blocSignalTest`

```dart
import 'package:bloc_signals_test/bloc_signals_test.dart';
import 'package:test/test.dart';

void main() {
  blocSignalTest<CounterBloc, int>(
    'emits 1 after Increment',
    build: () => CounterBloc(),
    act: (bloc) => bloc.add(Increment()),
    expect: () => [1],
  );
}
```

- `blocSignalTest<B, State>` takes `build`, `act`, `expect`, and optionally `wait:` for asynchronous handlers, plus `tags:`.
- Because handlers are synchronous, most tests need no `wait`.

## Direct assertions

```dart
test('synchronous emit updates stateValue immediately', () {
  final bloc = CounterBloc();
  bloc.add(Increment());
  expect(bloc.stateValue, 1);
  bloc.close();
});
```

Close every bloc you create in tests, including on failure (use `addTearDown(bloc.close)`).

## Asynchronous handlers

- Use `wait:` in `blocSignalTest`, or await the handler's future through a `Completer`, to observe results of `async` event handlers.
- `add` does not return the handler's future. Errors from async handlers are reported to the observer, and `close` does not cancel them.

## Widget tests

```dart
testWidgets('BlocSignalProvider + BlocSignalBuilder rebuilds on add', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: BlocSignalProvider(
        create: (_) => CounterBloc(),
        child: Builder(
          builder: (context) => Scaffold(
            body: BlocSignalBuilder<CounterBloc, int>(
              builder: (context, count) => Text('count=$count'),
            ),
            floatingActionButton: FloatingActionButton(
              onPressed: () => context.read<CounterBloc>().add(Increment()),
              child: const Icon(Icons.add),
            ),
          ),
        ),
      ),
    ),
  );
  expect(find.text('count=0'), findsOneWidget);
  await tester.tap(find.byType(FloatingActionButton));
  await tester.pump();
  expect(find.text('count=1'), findsOneWidget);
});
```

To replace a bloc in a widget test, use `BlocSignalProvider.value` with a fake instance you control.

## Commands

```bash
flutter test
flutter analyze
```
