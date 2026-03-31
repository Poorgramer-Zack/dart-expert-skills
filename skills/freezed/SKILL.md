---
name: "freezed-best-practices"
description: "Freezed code generation for immutable Dart data classes with copyWith, union types, and json_serializable integration. Use when creating data models, enforcing immutability, or generating fromJson/toJson serialization."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Freezed Latest Version Best Practices Guide (v3.2.x)

## Goal
Freezed is a highly popular code-generation package in Flutter utilized for fabricating Immutable data structures and Union Types. The current latest version is roughly **3.2.5**.

Even though Dart 3 natively supports Records and `sealed` classes, Freezed stubbornly retains an irreplaceable absolute advantage when handling complex API response models, deep copying (`copyWith`), and perfectly integrating with JSON serialization.

## Instructions

### 1. Dependencies and Environment Setup
Because Freezed relies heavily on code generation, ensure your `pubspec.yaml` is fully configured:

```yaml
dependencies:
  freezed_annotation: ^3.2.5
  json_annotation: ^4.9.0 # If mutual conversion with JSON is required

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^3.2.5
  json_serializable: ^6.8.0 # If mutual conversion with JSON is required
```

### 2. Defining an Immutable Data Class (Best Practices)
This is Freezed's most common utility. It automatically generates `==`, `hashCode`, `toString`, and the superbly useful `copyWith` for you.

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

// You MUST declare part files; the name must exactly match the current file
part 'user_model.freezed.dart';
part 'user_model.g.dart'; // If utilizing JSON serialization

// 🌟 Best Practices and Freezed 3.x Requirements: It is strongly recommended (and mandatory in specific scenarios) that the class invariably includes a `sealed` or `abstract` modifier.
// This ensures superior type safety, enables precise Pattern matching, and prohibits direct invalid instantiations.
@freezed
sealed class UserModel with _$UserModel {
  // Best Practice: Utilize `const factory` to declare constructors
  const factory UserModel({
    required String id,
    required String name,
    // Default values paired with the @Default annotation
    @Default(18) int age,
    // If identifying a JSON field name differs from the Dart variable name, utilize @JsonKey
    @JsonKey(name: 'is_active') @Default(true) bool isActive,
  }) = _UserModel;

  // Prerequisite for JSON Serialization
  factory UserModel.fromJson(Map<String, Object?> json) => 
      _$UserModelFromJson(json);
}
```

> **Crucial Tip**: Every time you modify the model, invariably execute `dart run build_runner build -d`.

### 3. Advanced Usage: Custom Getters and Methods
By default, the properties of a `@freezed` class are pure data. If you wish to append Computed properties or encapsulate distinct business logic methodologies, you **MUST append a private, parameterless constructor**.

```dart
@freezed
abstract class Product with _$Product {
  // 🌟 You MUST insert this private empty constructor; it notifies Freezed to authorize you to append custom internal methods safely
  const Product._();
  
  const factory Product({
    required double price,
    required double discountRate,
  }) = _Product;

  // Custom Getter
  double get discountedPrice => price * (1 - discountRate);

  // Custom Method
  bool isFree() => discountedPrice == 0;
}
```

### 4. Advanced Usage: Generic Types Support (Generic Types)
Freezed supports generics flawlessly, which is extraordinarily useful when architecting API Response wrappers or formulating universal lists.

```dart
@freezed
@JsonSerializable(genericArgumentFactories: true) // 🌟 This ensures Generics fully support JSON parsing logic
sealed class PaginatedResponse<T> with _$PaginatedResponse<T> {
  const factory PaginatedResponse({
    required int currentPage,
    required int totalPages,
    required List<T> data,
  }) = _PaginatedResponse<T>;

  // ⚠️ The fromJson implementation for generics is slightly unique; it mandates passing a specialized conversion function
  factory PaginatedResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Object? json) fromJsonT,
  ) => _$PaginatedResponseFromJson(json, fromJsonT);
}
```

### 5. Advanced Usage: Union Types (State Types) and Dart 3 Pattern Matching
Although Dart 3's native `sealed class` is phenomenally potent, when intersecting with legacy API architectures or when demanding the voluminous generation of state boilerplates at compile time, Freezed's Union Types remain fiercely versatile.

```dart
@freezed
sealed class ApiResponse with _$ApiResponse {
  const factory ApiResponse.loading() = ApiResponseLoading;
  const factory ApiResponse.data(List<String> items) = ApiResponseData;
  const factory ApiResponse.error(String message) = ApiResponseError;
}
```

**✅ Best Practice: Leverage Dart 3 Native Pattern Matching**
Freezed automatically declares subclasses, allowing it to seamlessly combine instantly with Dart 3's `switch` expressions:

```dart
Widget buildStatus(ApiResponse response) {
  return switch (response) {
    ApiResponseLoading() => const CircularProgressIndicator(),
    ApiResponseData(:final items) => Text('Item Count: ${items.length}'),
    ApiResponseError(:final message) => Text('Error: $message'),
  };
}
```
*💡 Note: The official documentation **strongly recommends** eliminating Freezed's legacy `.when()` or `.map()`, pivoting entirely toward native `switch` expressions to acquire superior compiler performance optimization and stringent exhaustion checking.*

### 6. Deep Copy (`copyWith`) vs `@unfreezed` Trade-offs

#### 6.1 Automatic Deep CopyWith
If your Data Class encapsulates another Freezed class (e.g., Company encompasses an Address class), Freezed will autonomously fabricate a deep chaining invocation API for you.

```dart
// Suppose `user` embraces property `address`, and `address` possesses `city`.
// Previously you penned: user.copyWith(address: user.address.copyWith(city: 'Taipei'));
// 🌟 Invoking Freezed's deep copyWith intuitively:
final newUser = user.copyWith.address(city: 'Taipei');
```

#### 6.2 When to unleash `@unfreezed`?
*   **Default Baseline**: It is vehemently advised to respect the core intentions of the package; **eternally use `@freezed` (immutable)**.
*   `@unfreezed` authorizes you to declare non-final variables, permitting direct destructive value modifications such as `user.age = 20`.
*   **👉 Best Practice Trade-offs**: This is categorically an **Anti-pattern** under strict State Management Architectures (like Riverpod/BLoC), as it utterly obliterates unidirectional data flow predictability constraints. Normally, use it prudently ONLY when aggressively wrestling with **hyper-frequently mutated colossal local forms (Forms) or isolated caches**, and exclusively when deep-copying incites indisputable phenomenological performance bottlenecks. Simultaneously, be aware it refuses to generate `==` and `hashCode` overrides (because the object inherently is mutable).

### 7. JSON Serialization Intricate Calibrations
If you routinely entail customizing parsing logic, brilliantly harness `@JsonKey`:

*   **Ignores specific fields entirely**: `@JsonKey(includeFromJson: false, includeToJson: false)`
*   **Customization Conversion Logic (Custom Converters)**: Confronting bizarre formats returned from backends (e.g., a date string formatted `"2024/01-01"` urgently demanding conversion to `DateTime`), you can forge customized `JsonConverter` subclasses:

```dart
class MyDateConverter implements JsonConverter<DateTime, String> {
  const MyDateConverter();
  @override
  DateTime fromJson(String json) => /* Custom parsing logic execution */;
  @override
  String toJson(DateTime object) => /* Revert string serialization logic */;
}

@freezed
sealed class Event with _$Event {
  const factory Event({
    @MyDateConverter() required DateTime eventDate, // Applying custom conversion annotation
  }) = _Event;
  // ... fromJson ...
}
```

### 8. Summary
1. Virtually all Data Transfer Objects (DTOs) and APP states (States) should be generated using Freezed.
2. Expand capabilities using private constructors to insert custom Methods, ensuring Data Classes elevate beyond simplistic structures.
3. Throw `.when()` resolutely into the trash bin; embrace the Dart 3 `switch` pattern matching paradigm exhaustively.
4. Adhere steadfastly to `@freezed` (Immutable), maintaining a rigid distance from `@unfreezed` to preserve architectural purity and impregnable safety.

## Constraints
* Ensure that utilizing generics within Freezed inevitably encompasses specifying `@JsonSerializable(genericArgumentFactories: true)` when implementing JSON translations.
* Completely prohibit `unfreezed` implementations inside all application-level state definitions, excepting ultra-niche granular massive-form handling architectures.
