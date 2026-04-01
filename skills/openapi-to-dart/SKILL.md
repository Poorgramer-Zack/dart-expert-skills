---
name: "converting-openapi-to-dart"
description: "Converts OpenAPI 3.0 specifications into null-safe Dart 3 models using Freezed and json_annotation, alongside Dio-based endpoint structures. Use this skill when generating Dart types from an OpenAPI file, converting a swagger spec to Dart models, creating Freezed classes from an API schema, automating type-safe API client creation, or any time the words 'openapi', 'swagger', 'generate dart', 'freezed models from spec', 'api contract', or 'openapi-to-dart' appear in the request. Handles $ref resolution, oneOf union types, enums, and request/response naming conventions."
metadata:
  last_modified: "2026-04-01 14:35:00 (GMT+8)"
---

# OpenAPI to Dart 3 (Effective Dart & Freezed)

Converts OpenAPI 3.0 specifications into robust, null-safe Dart 3 models leveraging `freezed` and `json_annotation`, alongside standardized Dio network endpoint structures.

**Input:** OpenAPI file (JSON or YAML)
**Output:** Dart file(s) containing Freezed models and API structures.

## When to Use

- "generate dart types from openapi"
- "convert openapi to dart"
- "create API models in dart"
- "generate freezed classes from spec"

## Workflow

1. Request the OpenAPI file path (if not provided).
2. Read and validate the file (must be OpenAPI 3.0.x).
3. Extract schemas from `components/schemas`.
4. Extract endpoints from `paths` (request/response types).
5. Generate Dart 3 files adhering to Effective Dart best practices.
6. Ask where to save (default: `lib/data/network/api.dart` or separated domain files).
7. Write the file(s).

## OpenAPI Validation

Check before processing:

```json
- Field "openapi" must exist and start with "3.0"
- Field "paths" must exist
- Field "components.schemas" must exist (if there are types)
```

If invalid, report the error distinctly and halt execution.

## Type Mapping (Dart 3 System)

### Primitives

| OpenAPI     | Dart 3       |
|-------------|--------------|
| `string`    | `String`     |
| `number`    | `double`     |
| `integer`   | `int`        |
| `boolean`   | `bool`       |
| `null`      | `Null`       |

### Format Modifiers

| Format        | Dart 3                  |
|---------------|-------------------------|
| `uuid`        | `String`                |
| `date`        | `DateTime`              |
| `date-time`   | `DateTime`              |
| `email`       | `String`                |
| `uri`         | `Uri` (or `String`)     |

### Complex Types (Freezed Execution)

**Object (Generating Freezed Data Classes):**
```dart
// OpenAPI: type: object, properties: {id, name}, required: [id]

import 'package:freezed_annotation/freezed_annotation.dart';

part 'example.freezed.dart';
part 'example.g.dart';

@freezed
sealed class Example with _$Example {
  const factory Example({
    required String id,      // required: enforce non-nullable & required
    String? name,            // optional: omit required, make nullable `?`
  }) = _Example;

  factory Example.fromJson(Map<String, dynamic> json) => _$ExampleFromJson(json);
}
```

**Array (Lists):**
```dart
// OpenAPI: type: array, items: {type: string}
// Represented as `List<String>` wherever consumed.
typedef Names = List<String>;
```

**Enum (Enhanced Dart 3 Enums):**
```dart
// OpenAPI: type: string, enum: [active, draft]
@JsonEnum(fieldRename: FieldRename.snake)
enum Status {
  active,
  draft,
}
```

**oneOf (Union Types mapped via Freezed):**
```dart
// OpenAPI: oneOf: [{$ref: Cat}, {$ref: Dog}]
@freezed
sealed class Pet with _$Pet {
  const factory Pet.cat(Cat data) = _PetCat;
  const factory Pet.dog(Dog data) = _PetDog;
  
  factory Pet.fromJson(Map<String, dynamic> json) => _$PetFromJson(json);
}
```

## Code Generation Strict Guidelines

### File Header

```dart
// Auto-generated from: {source_file}
// Generated at: {timestamp}
//
// DO NOT EDIT MANUALLY - Regenerate from OpenAPI schema
```

### Models (from components/schemas)

For each schema in `components/schemas`, utilize `@freezed`:

```dart
@freezed
sealed class Product with _$Product {
  const factory Product({
    /// Product unique identifier
    required String id,

    /// Product title
    required String title,

    /// Product price
    required double price,

    /// Created timestamp. Map from snake_case automatically using JsonSerializable.
    @JsonKey(name: 'created_at') DateTime? createdAt,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) => _$ProductFromJson(json);
}
```

*   **Effective Dart Mapping**: Default properties to `lowerCamelCase` inside the Freezed class. Use `@JsonKey(name: '...')` if the OpenAPI spec provides `snake_case` or `kebab-case`.
*   **Documentation**: Transport OpenAPI `description` attributes directly into Dart triple-slash `///` doc comments.
*   **Immutability**: Freezed classes natively satisfy Effective Dart's immutability mandates avoiding unwarranted explicit Setters.

### Request/Response Endpoint Structures (Dio Blueprint)

For each endpoint inside `paths`, generate clear structures.

```dart
// GET /products - query params
@freezed
sealed class GetProductsRequest with _$GetProductsRequest {
  const factory GetProductsRequest({
    int? page,
    int? limit,
  }) = _GetProductsRequest;

  factory GetProductsRequest.fromJson(Map<String, dynamic> json) => _$GetProductsRequestFromJson(json);
}

// GET /products - response 200
typedef GetProductsResponse = List<Product>;

// POST /products - request body
@freezed
sealed class CreateProductRequest with _$CreateProductRequest {
  const factory CreateProductRequest({
    required String title,
    required double price,
  }) = _CreateProductRequest;

  factory CreateProductRequest.fromJson(Map<String, dynamic> json) => _$CreateProductRequestFromJson(json);
}

// POST /products - response 201
typedef CreateProductResponse = Product;
```

Naming convention mandate:
*   `{Method}{Path}Request` for params/body mappings.
*   `{Method}{Path}Response` for responses.

### Error Type Handling (Global Catch)

Always furnish a baseline error catching model complying with the API's global error formats.

```dart
@freezed
sealed class ApiError with _$ApiError {
  const factory ApiError({
    required int status,
    required String error,
    String? detail,
  }) = _ApiError;

  factory ApiError.fromJson(Map<String, dynamic> json) => _$ApiErrorFromJson(json);
}
```
*Note: Type guards (like `isProduct` in TypeScript) are intrinsically navigated in Dart through native `is` keyword checking (e.g., `if (response is ApiError)`) coupled with Dart 3 exhaustive pattern matching utilizing `sealed` classes or unified Results.*

## $ref Resolution Strategy

Upon discovering `{"$ref": "#/components/schemas/Product"}`:
1. Extract the class terminology (`Product`).
2. Implement the type reference directly preventing inline resolutions duplicating structures recursively.

```dart
// OpenAPI: items: {$ref: "#/components/schemas/Product"}
// Dart:
final List<Product> items;
```

## Constraints & Effective Dart Adherence
*   `UpperCamelCase` for Classes/Enums/Typedefs. `lowerCamelCase` for properties and functions.
*   Enforce `required` directives rigorously eliminating null-assertion operator (`!`) risks entirely out of the pipeline.
*   Never manually forge `.fromJson` parsers string-by-string; exclusively generate `@freezed` + `@JsonSerializable` boilerplates cleanly supporting `json_serializable` ecosystem standards securely.
*   **Freezed 3.0 Mandate**: All generated `@freezed` classes must explicitly be declared as `sealed` or `abstract` (prioritize `sealed class`) to satisfy compiler constraints mapping correctly with Freezed 3.0+.
*   If an Unknown OpenAPI type is encountered, fallback to Dart's `dynamic` but firmly warn the user.
