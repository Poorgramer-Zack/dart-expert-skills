---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Serverpod Endpoints and Database Operations (Endpoints & Database)

## Goal
Implements Serverpod API interfaces via Endpoints. Any public method defined in an Endpoint, provided it follows specific typing rules, is automatically compiled into a Dart function that the frontend Client can call directly.

## Instructions

### Endpoints and the Session Context
All Endpoint classes must inherit from `Endpoint` and be placed inside `<project>_server/lib/src/endpoints/`.

#### The Core: Session Object
**The first parameter of every public method MUST be a `Session` object**.
The `Session` contains the entire context of a Request's lifecycle. It is your gateway to calling various core services on the backend:

- `session.db`: Database operations
- `session.caches`: Access Redis or Local caches
- `session.messages`: Server internal Event passing system
- `session.authenticated`: Identifies who is currently calling the API (Requires the Auth module)
- `session.log()`: Natively integrated logging

```dart
import 'package:serverpod/serverpod.dart';
import '../generated/protocol.dart';

class ArticleEndpoint extends Endpoint {
  
  // 🌟 Methods exposed to the frontend MUST return a Future
  Future<Article> getArticle(Session session, int id) async {
    // Access all backend resources through the session here
    session.log('Fetching article $id', level: LogLevel.info);
    
    // Example: Fetching from the database
    var article = await Article.db.findById(session, id);
    if (article == null) {
      // 🌟 Exceptions can be thrown directly, and the Client side can catch them
      throw Exception('Article not found'); 
    }
    return article;
  }
}
```

### Database Queries (Database Queries / ORM)
Serverpod provides an extremely Type-safe ORM. You **do not need to write SQL statements**; all `Where` conditions are checked at compile time.

#### Basic CRUD
When operating on the database, always use the static `.db` calls of the generated classes:

```dart
// Insert
await UserProfile.db.insertRow(session, UserProfile(name: 'Zack', email: 'a@a.com'));

// Read a single record (FindById)
var user = await UserProfile.db.findById(session, 1);

// Update
user.name = 'Zack Updated';
await UserProfile.db.updateRow(session, user);

// Delete
await UserProfile.db.deleteRow(session, user);
```

#### Complex Where Queries
Use `find` to get a list, and use the generated field constants (e.g., `t.age`) to assemble conditions:

```dart
var users = await UserProfile.db.find(
  session,
  // 🌟 Incredibly powerful strongly-typed Where conditions
  where: (t) => t.age > 18 & t.name.like('%Zack%') | t.age.equals(null),
  orderBy: (t) => t.joinedDate,
  orderDescending: true,
  limit: 10,
  offset: 0,
);
```

#### Relational Queries (Include)
When you need to pull related data simultaneously, use `include` (one-to-one) or `includeList` (one-to-many):

```dart
var companies = await Company.db.find(
  session,
  // 🌟 Join and pull all employees of the company simultaneously
  include: Company.include(
    employees: Employee.includeList(),
  ),
);

// companies[0].employees can now be safely accessed!
```

### Database Transactions
When your business logic requires an "All-or-Nothing" guarantee (e.g., transfer operations, simultaneously creating multiple related orders), you must use a Transaction.

#### Implementing a Transaction
Call `session.db.transaction()` and pass the obtained `transaction` object forward to all CRUD operations:

```dart
Future<bool> createOrder(Session session, Order order, List<OrderItem> items) async {
  try {
    // 🌟 Open a transaction
    await session.db.transaction((transaction) async {
      
      // 1. Write the main order (Note: The transaction parameter is added at the end)
      await Order.db.insertRow(session, order, transaction: transaction);
      
      // 2. Write multiple sub-items
      for (var item in items) {
        item.orderId = order.id;
        await OrderItem.db.insertRow(session, item, transaction: transaction);
      }
      
      // 3. Deduct inventory (Virtual logic)
      // await Inventory.db.updateRow(session, ... , transaction: transaction);
      
      // 🌟 If any exception is thrown midway, the entire data set will automatically rollback!
    });
    return true;
  } catch (e) {
    session.log('Order creation failed', exception: e);
    return false;
  }
}
```

## Constraints
* Ensure all Endpoint method signatures contain `Session session` as the first argument.
* Never manually pass backend objects out to the client if they are not defined within the `PROTOCOL` YAMLs. Use `throw Exception()` (or custom Exceptions defined in YAML) to handle errors cleanly.
