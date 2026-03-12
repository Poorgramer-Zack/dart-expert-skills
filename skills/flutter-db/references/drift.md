---
name: "drift-best-practices"
description: "Best practices and guidelines for Drift Latest Version Best Practices Guide (v2.32.x). Use this skill when you need to write or review code related to Drift Latest Version Best Practices Guide (v2.32.x)."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Drift Latest Version Best Practices Guide (v2.32.x)

## Goal
`drift` is the pinnacle, entirely Type-safe relational database (SQLite) solution within the Flutter ecosystem. The latest version is roughly **2.32.0**, and it is highly recommended to implement it seamlessly utilizing the official native helper package `drift_flutter` (v0.3.0).

## Instructions

### 1. Core Concepts and Dependency Installation
Drift’s immense power lies in writing database table structures (Tables) in Dart or authoring native SQL, and subsequently leveraging code generation to acquire all type-safe query methods and `DataClass`es.

#### Dependency Setup
```yaml
dependencies:
  drift: ^2.32.0
  drift_flutter: ^0.3.0

dev_dependencies:
  drift_dev: ^2.32.0
  build_runner: ^2.4.0
```

### 2. Two Methods for Defining Tables (Best Practices)
You can opt to use **Dart Classes** or write **`.drift` native SQL files**.
> ✅ **Best Practice**: For simple to moderately complex projects, employ **Dart Classes**; if your project encompasses a massive quantity of complex JOIN clauses or massive native SQL migrations, utilize **`.drift` files**. This guide primarily focuses on Dart definitions.

#### Step A: Defining Tables (Tables)
All custom tables must inherit from `Table`.

```dart
import 'package:drift/drift.dart';

class Todos extends Table {
  // Primary key auto-increment
  IntColumn get id => integer().autoIncrement()();
  
  // Text constraints
  TextColumn get title => text().withLength(min: 1, max: 50)();
  
  // Supports nullability and default values
  TextColumn get content => text().nullable()();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
  
  // Foreign key example (Assuming a Categories table exists)
  // IntColumn get categoryId => integer().nullable().references(Categories, #id)();
}
```

#### Step B: Configuring the Database and Initialization
Utilize the newly introduced `drift_flutter` to conquer lengthy native connection setups.

```dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';

// part file generation target
part 'app_database.g.dart';

@DriftDatabase(tables: [Todos]) 
class AppDatabase extends _$AppDatabase {
  
  // Best Practice: Utilize driftDatabase provided by drift_flutter to initialize cross-platform connections
  AppDatabase() : super(driftDatabase(name: 'my_app_db'));

  @override
  int get schemaVersion => 1; // You MUST increment this tightly every time you modify the table structure
}
```
**Compilation**: Execute `dart run build_runner build -d`.

### 3. Database Operations: CRUD (Strongly Typed)
The generated `app_database.g.dart` establishes a data class incorporating all columns (e.g., `Todo`, removing the 's').

```dart
// Append these methods within the AppDatabase class:

// 1. Read
Future<List<Todo>> getAllTodos() => select(todos).get();

// 1-1. Read and return as a Stream (Tremendous reactivity! When the DB changes, the UI updates automatically)
Stream<List<Todo>> watchAllTodos() => select(todos).watch();

// 1-2. Conditional Filtering (Where)
Future<List<Todo>> getDailyTodos() {
  return (select(todos)..where((t) => t.title.like('%Daily%'))).get();
}

// 2. Create (Write)
Future<int> insertTodo(TodosCompanion todo) => into(todos).insert(todo);

// 3. Update
Future<bool> updateTodo(Todo todo) => update(todos).replace(todo);

// 4. Delete
Future<int> deleteTodo(int id) {
  return (delete(todos)..where((t) => t.id.equals(id))).go();
}
```

> **Crucial Tip: What exactly is `TodosCompanion`?**
When Inserting data, if a column is auto-incrementing (like ID) or possesses a default value (like created_at), you cannot directly pass an entire `Todo` class instance (because `Todo` demands that all non-null columns be populated). Utilizing the compiler-generated `TodosCompanion` effortlessly resolves partial column updates or insertion requirements.

```dart
// No need to supply id and createdAt during insertion
db.insertTodo(TodosCompanion.insert(
  title: 'Do laundry',
  content: const Value('Remember to buy detergent'),
));
```

### 4. Splitting Logic using DAOs (Data Access Objects)
When your database orchestrates dozens of Tables, writing every CRUD operation inside `AppDatabase` morphs into a maintainability catastrophe.
**Best Practice**: Employ DAOs to partition modules robustly.

```dart
// 1. Declare which tables the DAO governs
@DriftAccessor(tables: [Todos])
class TodosDao extends DatabaseAccessor<AppDatabase> with _$TodosDaoMixin {
  // Inject database
  TodosDao(AppDatabase db) : super(db);

  // Author exclusive Todos CRUD methods here...
  Stream<List<Todo>> watchAll() => select(todos).watch();
}

// 2. Register the DAO back into AppDatabase
@DriftDatabase(tables: [Todos], daos: [TodosDao])
class AppDatabase extends _$AppDatabase { ... }
```

### 5. Optimizing Performance with Background Threads (Isolates)
When handling colossal data insertions or heavy queries, it triggers UI jank (stuttering) in Flutter.
Drift natively supports executing all DB operations within an Isolate.

Through `drift_flutter`, enabling background services requires merely a single parameter configuration:
```dart
AppDatabase() : super(driftDatabase(
  name: 'my_app_db',
  // Activate this setting, and Drift automatically offloads the connection to a Background Isolate for execution,
  // completely preempting database access from blocking the main UI Thread.
  web: DriftWebOptions(
    sqlite3Wasm: Uri.parse('sqlite3.wasm'),
    driftWorker: Uri.parse('drift_worker.dart.js'),
  ),
));
```

### 6. Summary
1. **Rely on `drift_flutter`**: For new projects, directly incorporate `drift_flutter` to establish connections, drastically mitigating cross-platform configuration friction.
2. **Adopt Stream-Driven UI**: Skillfully deploy `.watch()` to return Streams. Paired with state management tools like StreamBuilder or Riverpod, you can forge exceptionally smooth responsive Offline-first experiences.
3. **Isolate Responsibilities employing DAOs**: For substantial projects, absolutely harness `@DriftAccessor` to shepherd database operational logic efficiently.

## Constraints
* Assure you ALWAYS execute code generation `dart run build_runner build -d` traversing any Drift table or query modification.
* Strictly bind `drift_flutter` for database initialization cross-platform compatibilities instead of writing native FFI initialization procedures manually.
