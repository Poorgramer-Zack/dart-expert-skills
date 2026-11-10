# Error handling with Either (dart_either)

Replaces the old `fpdart` guidance. Verified against `dart_either` 2.5.0 with `dart analyze` and `dart run`.

## When to use Either

- Expected failures (validation, not found, network) that callers must handle. Use `Either<Failure, T>`.
- Keep exceptions for bugs and programmer errors. Wrap throwing third-party code once, at the boundary, with `Either.tryCatch` / `Either.tryCatchAsync`.
- Do not wrap every function. Use `Either` where the caller benefits from being forced to handle the failure.

## Setup

```yaml
dependencies:
  dart_either: ^2.5.0
```

```dart
import 'package:dart_either/dart_either.dart';
```

## Model the failure as a sealed type

```dart
sealed class UserError {
  const UserError();
}

final class NotFound extends UserError {
  const NotFound(this.id);
  final int id;
}

final class NetworkError extends UserError {
  const NetworkError(this.message);
  final String message;
}
```

## Construct and chain

```dart
Either<UserError, int> parseId(String raw) {
  final id = int.tryParse(raw);
  return id == null ? const Either.left(NetworkError('bad id')) : Either.right(id);
}

Either<UserError, String> findName(int id) =>
    id == 1 ? const Either.right('Ada') : Either.left(NotFound(id));

final a = parseId('1').flatMap(findName).map((name) => name.toUpperCase()); // Right(ADA)
final b = parseId('2').flatMap(findName);                                   // Left(NotFound(2))
```

- `Either.left(...)` and `Either.right(...)` are the constructors.
- `flatMap` chains a function that returns an `Either`. `map` transforms the right value. `mapLeft` transforms the failure.

## Handle the result

```dart
final message = b.fold(
  ifLeft: (e) => switch (e) {
    NotFound(:final id) => 'no user $id',
    NetworkError(:final message) => 'network: $message',
  },
  ifRight: (name) => 'hello $name',
);
```

- Prefer `fold` (or a `switch` on `isLeft` / `isRight`) when both branches matter.
- `getOrNull()`, `getOrDefault(value)`, `getOrHandle((l) => ...)`, and `getOrThrow()` are available. `getOrElse` is deprecated.

## Async

```dart
Future<Either<UserError, String>> fetchName(int id) async => findName(id);

// Transform the value inside a Future<Either>
final length = await fetchName(1).thenMapEither((name) => name.length); // Right(3)

// Chain another Future<Either>
final greeting = await fetchName(1).thenFlatMapEither((name) async => Either.right('hi $name'));
```

- `thenMapEither` and `thenFlatMapEither` are extension methods on `Future<Either<L, R>>`. They are not defined on `Either` itself, so you cannot call them on a plain `Either`.

## Wrap throwing code

```dart
final parsed = Either<Object, int>.tryCatch(
  action: () => int.parse('x'),
  errorMapper: (error, stackTrace) => error,
);

final remote = await Either.tryCatchAsync<Object, int>(
  action: () async => int.parse('x'), // any throwing async call
  errorMapper: (error, stackTrace) => error,
);
```

Map the caught error to your own `Failure` type in `errorMapper`. Do not let raw exceptions leak into the domain layer.

## Do-notation with `bindingAsync`

Use `Either.bindingAsync` when several `Either` steps depend on each other. The first `Left` stops the block.

```dart
final result = await Either.bindingAsync<UserError, String>((effect) async {
  final id = effect.bind(parseId('1'));
  final name = effect.bind(await fetchName(id));
  return '$name!';
});
```

- Use `effect.bind(either)` for sync values and `effect.bind(await future)` for futures.
- The `effect` object is only valid inside its own block. Do not store it or call it after the block finishes.

## Rules

- Keep the `Failure` type small and domain-specific. Do not use `String` as the only error type.
- Convert errors at the data-layer boundary, then pass `Either` up. The UI decides how to show each failure.
- Do not mix `Either` with `throw` for the same kind of failure in one function.
