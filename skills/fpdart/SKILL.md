---
name: "fpdart"
description: "Functional programming library for Dart and Flutter. Replaces exceptions with Either, handles nulls with Option, and enables composable async logic with Task and TaskEither."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Functional Programming with fpdart

## Goal
Implement robust, type-safe, and predictable business logic using functional programming principles. Replace side-effect-prone imperative code with pure functions and composable monads (`Option`, `Either`, `Task`), ensuring that error handling and null-safety are enforced at the type level rather than through runtime exceptions.

## Process

### Phase 1: Basic Data Containment
Use primitives to handle common edge cases without branching logic:
- **`Option<T>`**: Replaces `T?`. Use `Option.of(val)` or `Option.none()`. Access via `match` or `getOrElse`.
- **`Either<L, R>`**: Replaces `try-catch`. `L` (Left) is for errors, `R` (Right) is for success.
- **`Unit`**: Replaces `void` to allow functional composition.

### Phase 2: Execution & Async Flow
Manage side effects and asynchronous operations through wrappers that defer execution:
- **`IO<T>`**: Synchronous wrapper for pure/safe side effects.
- **`Task<T>`**: Asynchronous wrapper for `Future` that never fails.
- **`TaskEither<L, R>`**: The standard for API requests. Composes `Future` and `Either` into a single monadic flow.

### Phase 3: Advanced Linear Chaining
Utilize the **Do Notation** to flatten nested `flatMap` calls into readable, imperative-style blocks while preserving monadic safety:
```dart
return TaskEither.Do(($) async {
  final user = await $(repository.getUser(id));
  final profile = await $(api.getProfile(user.code));
  return profile.toDomain();
});
```

---

## ⚡ Quick Reference

### Error Handling (Either)
```dart
Either<String, int> checkValue(int v) => 
  v > 10 ? Either.of(v) : Either.left('Too small');

final result = checkValue(5).match(
  (error) => 'Failure: $error',
  (value) => 'Success: $value',
);
```

### Async Operations (TaskEither)
```dart
TaskEither<Exception, User> fetchUser(String id) =>
  TaskEither.tryCatch(
    () => api.getUser(id),
    (error, stack) => Exception('Fetch failed: $error'),
  );
```

### Collections Extension
`fpdart` extends `Iterable` with safer methods:
- `list.head` -> `Option<T>` (Safe `first`)
- `list.all((e) => ...)` -> `bool`

## Constraints
- **Zero Exceptions**: Prohibit the use of `throw` for expected business errors. Force use of `Either` or `TaskEither`.
- **Monadic Integrity**: Never use `!` (bang operator) or force-casts on functional types. Always use safe extraction methods (`match`, `getOrElse`).
- **Linear Logic**: Mandatory use of **Do notation** when chaining more than two monadic operations to prevent "callback hell."
- **Immutability First**: Combine with `fast_immutable_collections` for all state definitions.
