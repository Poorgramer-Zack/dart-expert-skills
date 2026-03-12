---
name: "dart-best-practices"
description: "Best practices and guidelines for Dart 3 Latest Features and Effective Dart Best Practices Guide. Use this skill when you need to write or review code related to Dart 3 Latest Features and Effective Dart Best Practices Guide."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Dart 3 Latest Features and Effective Dart Best Practices Guide

## Goal
This document amalgamates the powerful latest features of `Dart 3` with official `Effective Dart` best practices, equipping you with a comprehensive guide for writing highly efficient, safe, and maintainable Dart code.

## Instructions

### 1. Core Focus: Dart 3 Latest Features and Application Best Practices
Dart 3 introduces numerous revolutionary syntax and type system upgrades, strictly enforcing Sound Null Safety comprehensively.

#### 1.1 Records
Records permit the aggregation of multiple values of diverse types into a single object without the necessity of declaring a dedicated Class.
* **Best Practices**:
  * **Multiple Return Values**: When a function needs to return multiple values, prioritize Records rather than constructing throwaway wrapper classes (Data Classes).
  * **Named Fields**: When returning more than two values or when semantics are ambiguous, utilize named fields to elevate readability.
  ```dart
  // ✅ Recommended Usage
  ({double lat, double lon}) getLocation(String city) {
    return (lat: 25.0330, lon: 121.5654);
  }
  ```

#### 1.2 Patterns and Destructuring
Patterns can scrutinize and synchronously destructure data, drastically simplifying attribute extraction from Records, Lists, Maps, and custom objects.
* **Best Practices**:
  * **Destructuring during variable declaration**: If you need to extract values from a Record or List, destructure them directly upon declaration.
  * **Replacing complex if statements**: Employ the `if-case` syntax to concurrently validate formatting and extract variables.
  ```dart
  // ✅ Recommended Usage
  final json = {'user': ['Zack', 25]};
  if (json case {'user': [String name, int age]}) {
    print('User $name is $age years old.');
  }
  ```

#### 1.3 `switch` Expressions and Exhaustive Checking
Dart 3 introduces `switch` expressions (superseding portions of switch statements) and provides compile-time exhaustive checking for specific types (e.g., `sealed` classes, Enums).
* **Best Practices**:
  * **Assignment as an expression**: When you need to return diverging values based on different conditions, use Switch expressions to make the code far more concise.
  * **Omit `default`**: If you have already exhaustively addressed an Enum or `sealed` class, do not append a `default` branch. Thus, when new types are appended in the future, the compiler will error out and remind you to update your logic.

#### 1.4 Class Modifiers (Class Modifiers)
Dart 3 introduced modifiers like `sealed`, `interface`, `base`, and `final`, allowing you to control class inheritance and implementation with heightened granularity.
* **Best Practices**:
  * **`sealed`**: Utilized for constructing Algebraic Data Types (ADTs). Subclasses within the same library can inherit from it, while external inheritance is obstructed. Paired with `switch`, it forces exhaustive checking. This is phenomenally well-suited for defining States or Result/Error types.
  * **`interface`**: Utilize this when you exclusively desire the class to act as a contract via `implements`, but do not want its implementation details inherited via `extends`.
  * **`final`**: Completely prohibiting external libraries from inheriting (`extends`) or implementing (`implements`), ensuring the defensive integrity of the class.

### 2. Effective Dart: Style and Formatting (Style)
Maintaining consistent code aesthetic is the bedrock of collaboration.

* **Linters and Formatting**: It is strongly recommended to use `dart format` to auto-format code, and to enable officially recommended `core` or `recommended` lints inside `analysis_options.yaml`.
* **Naming Conventions**:
  * **Classes, Enums, Typedefs, Type parameters**: Use `UpperCamelCase` (Pascal Case).
  * **Libraries, Packages, Directories, Source files**: Use `lowercase_with_underscores` (Snake Case).
  * **Variables, Parameters, Functions, Methods**: Use `lowerCamelCase`.
  * **Constants**: Please utilize `lowerCamelCase` (e.g.: `const defaultTimeout = 1000;`), and refrain from using fully capitalized letters spliced with underscores (SCREAMING_CAPS).

### 3. Effective Dart: Programming and Usage (Usage)

#### 3.1 Variables and Types
* **Type Inference**: Let the compiler deduce types; omit type annotations whenever practically possible. For local variables, abundantly employ `var` or `final`.
* **Collection Usage (Collections)**:
  * Initialize collections using Literals: `var list = [];`, `var map = {};`. Avoid utilizing the `List()` or `Map()` constructor invocations.
  * Leverage **Spread operators (`...`)** and **collection control flow (`if`, `for`)** to construct dynamic Lists/Maps; this is sizably more concise than repeatedly calling `.add()`.
* **Lazy Initialization**: Utilize `late` exclusively when calculation delay is fundamentally required, or when an object isn't null but the initialization flow occurs immediately following construction. Meticulously prevent accessing `late` variables prior to assignment to avert runtime trauma.

#### 3.2 Functions and Asynchrony (Functions & Async)
* **Arrow Functions (`=>`)**: When a function contains solely a singular return expression, proficiently utilize arrow functions to condense the code.
* **Parameter Design**:
  * Confronting multiple potentially optional parameters, prioritize the use of **Named parameters** and affix `required` per necessity, clarifying the caller's intent immensely.
* **Futures and Asynchrony**:
  * Prioritize establishing flows using `async` / `await` over manually chaining `.then()` and `.catchError()`. This increases legibility and greatly assists the deployment of traditional `try/catch` enclosures.

#### 3.3 Null Safety
* **Minimize nullable types**: Ascertain designs where variables categorically cannot be null. Utilize `T?` merely when a value genuinely might lack existence.
* **Avoid abusing the `!` (Bang operator)**: Attempt aggressively to manage nulls via Type Promotion mechanisms like `if (value != null)` or Pattern matching, drastically minimizing potential crash liabilities instigated by forced unwrapping.

### 4. Effective Dart: API Design (Design)
* **Keep Interfaces pristine**: Only expose members requiring exterior invocation. Dart defaults to public; if a variable or function mandates privacy, prefix its namespace with an underscore `_`.
* **Harness Getter and Setters to supersede rudimentary operational functions**:
  * Use `get` if the function's activity resembles retrieving a property, free of ostensible side effects or exorbitant calculation outlays.
  * Do not blindly provide a setter for every property; prioritize assigning properties as `final` during constructor initialization (Immutable Data design concept).
* **Constructor Design**:
  * Exploit Dart's Redirecting constructors or Named constructors (e.g., `User.fromJson`) aggressively to upgrade code semantics.
  * Whenever imaginable, provide `const` constructors, assisting the compiler massively in executing memory and performance optimization routines.

## Constraints
* Heavily apply `Records` when confronted with operations demanding composite returns to eradicate the scourge of one-off Data classes.
* Never explicitly type variables where the Dart compiler can intuitively execute Type Inference securely.
