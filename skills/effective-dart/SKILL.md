---
name: "applying-effective-dart"
description: "Dart 3 modern features and Effective Dart best practices including records, patterns, sealed classes, and extension types. Use when adopting Dart 3 syntax, improving code quality, or reviewing Dart code conventions."
metadata:
  last_modified: "2026-04-01 14:35:00 (GMT+8)"
---

# Dart 3 Latest Features and Effective Dart Best Practices Guide

## Goal
Covers Dart 3 features and Effective Dart practices for writing safe, maintainable code.

## Instructions

### 1. Dart 3 Features

#### 1.1 Records
Records aggregate multiple typed values without requiring a dedicated class.
* **Best Practices**:
  * **Multiple Return Values**: Prefer Records over throwaway wrapper classes.
  * **Named Fields**: Use named fields when returning more than two values or when semantics are ambiguous.
  ```dart
  // ✅ Recommended Usage
  ({double lat, double lon}) getLocation(String city) {
    return (lat: 25.0330, lon: 121.5654);
  }
  ```

#### 1.2 Patterns and Destructuring
Patterns match and destructure data from Records, Lists, Maps, and custom objects.
* **Best Practices**:
  * **Destructuring on declaration**: Extract values from Records or Lists directly at assignment.
  * **Replace complex if-statements**: Use `if-case` to validate shape and extract variables simultaneously.
  ```dart
  // ✅ Recommended Usage
  final json = {'user': ['Zack', 25]};
  if (json case {'user': [String name, int age]}) {
    print('User $name is $age years old.');
  }
  ```

#### 1.3 `switch` Expressions and Exhaustive Checking
`switch` expressions return values and provide compile-time exhaustiveness for sealed classes and enums.
* **Best Practices**:
  * **Use as expression**: Assign diverging values concisely with switch expressions.
  * **Omit `default`**: On sealed/enum types, omit `default` so the compiler catches unhandled cases when new variants are added.

#### 1.4 Class Modifiers
Dart 3 class modifiers control inheritance and implementation boundaries.
* **Best Practices**:
  * **`sealed`**: For Algebraic Data Types (ADTs). Enables exhaustive switch checking. Ideal for State or Result/Error types.
  * **`interface`**: Allows `implements` only; prevents `extends`.
  * **`final`**: Prevents both `extends` and `implements` from external libraries.

### 2. Effective Dart: Style
Consistent style enables collaboration.

* **Formatting**: Use `dart format` and enable `core` or `recommended` lints in `analysis_options.yaml`.
* **Naming Conventions**:
  * **Classes, Enums, Typedefs, Type parameters**: `UpperCamelCase`
  * **Libraries, Packages, Directories, Files**: `lowercase_with_underscores`
  * **Variables, Parameters, Functions, Methods**: `lowerCamelCase`
  * **Constants**: `lowerCamelCase` (e.g., `const defaultTimeout = 1000;`) — not SCREAMING_CAPS.

### 3. Effective Dart: Usage

#### 3.1 Variables and Types
* **Type Inference**: Use `var`/`final` for local variables; omit type annotations where the compiler infers correctly.
* **Collections**: Initialize with literals (`var list = [];`). Use spread operators and collection `if`/`for` instead of `.add()` calls.
* **`late`**: Use only when initialization must be deferred. Never access before assignment.

#### 3.2 Functions and Async
* **Arrow Functions**: Use `=>` for single-expression functions.
* **Named Parameters**: Prefer named parameters for multiple optional arguments; apply `required` as needed.
* **Async**: Prefer `async`/`await` over `.then()`/`.catchError()` chains.

#### 3.3 Null Safety
* **Minimize nullables**: Design variables as non-nullable where possible.
* **Avoid `!`**: Use type promotion (`if (value != null)`) or pattern matching instead of forced unwrapping.

### 4. Effective Dart: API Design
* **Minimal exposure**: Only expose members that require external access. Prefix private members with `_`.
* **Getters/Setters**: Use `get` for property-like reads with no side effects or expensive computation. Don't add setters for every property — prefer `final` constructor initialization.
* **Constructors**: Use named constructors (e.g., `User.fromJson`) and redirecting constructors for semantics. Provide `const` constructors wherever possible.

## Constraints
* Use `Records` for composite returns instead of one-off data classes.
* Omit type annotations where the compiler can infer the type safely.
