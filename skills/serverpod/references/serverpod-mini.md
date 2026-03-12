---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Serverpod Mini: The Extremely Fast Database-less Backend Experience

## Goal
Implements a lightweight Serverpod backend without a database. Sometimes we just want to write a simple API (e.g., to connect to a third-party service, or return Mock data), or the team is still in the Proof of Concept (PoC) stage. **Serverpod Mini** is the best choice for this.

## Instructions

### What is Serverpod Mini?
Serverpod Mini is a "lightweight, stripped-down version" of Serverpod.
It removes the dependency on PostgreSQL, allowing you to launch it with one click **without setting up a database**.
The best part is that its API protocols and YAML model definitions are completely identical to the full version. In the future, when the project needs to grow, **it can seamlessly upgrade to a full Serverpod at any time.**

### Creating a Mini Project
Add the `--mini` parameter in the terminal to create it:

```bash
serverpod create myminipod --mini
```

Upon creation, you will get three directories as usual:
1. `myminipod_server`: Your Dart server source code.
2. `myminipod_client`: Automatically generated, type-safe API client.
3. `myminipod_flutter`: A Flutter example project with the client already wired up.

#### Starting the Server
As with the standard version, navigate to the server directory and run main:
```bash
cd myminipod/myminipod_server
dart bin/main.dart
```
*(You do not need to start docker-compose to run postgres first; it revives instantly!)*

### Defining Models and Data Transfer (Defining Models)
In the Mini version, you still enjoy the benefits of YAML forced contracts. Create `.spy.yaml` anywhere under `myminipod_server/lib/`.

**Example**:
```yaml
class: Company
# 🌟 Note: The Mini version omits the "table: company" declaration line because it doesn't bind to a database.
fields:
  name: String
  foundedDate: DateTime?
  employees: List<String>
```

It supports various primitive types (including `DateTime`, `UuidValue`, `BigInt`, etc.) and even `Record`.
After saving, always remember to run:
```bash
serverpod generate
```

### Writing and Calling Endpoints
After defining the Model, you can write business logic in the `endpoints/` directory.

```dart
// myminipod_server/lib/src/endpoints/company_endpoint.dart
import 'package:serverpod/serverpod.dart';
import '../generated/protocol.dart';

class CompanyEndpoint extends Endpoint {
  
  // Simulate an API that returns detailed company information
  Future<Company> getCompanyDetails(Session session, String name) async {
    // This is typically where you would call a third-party API, or read from a file/memory
    return Company(
      name: name,
      foundedDate: DateTime.now(),
      employees: ['Alice', 'Bob', 'Charlie'],
    );
  }
}
```

Save and run `serverpod generate` again.

**Flutter Call**:
Call it extremely fast in Flutter via the automatically generated Client:
```dart
final company = await client.company.getCompanyDetails('Google');
print(company.employees.first); // Alice
```

### Conclusion and Suitable Scenarios

**Advantages**:
- **Zero Configuration**: No need to fiddle with Docker and Postgres; develop in seconds.
- **Type Safety**: Still retains end-to-end type protection from YAML to Dart.
- **Seamless Upgrade**: A Mini project can be converted into a regular Serverpod requiring a database at any time.

**Suitable Scenarios**:
- **BFF (Backend for Frontend)**: Aggregate complex external third-party APIs here, then spit out clean Models to Flutter.
- **Microservices**: If certain server nodes do not need to store persistent data and are purely for CPU computation (e.g., image processing), Mini is the perfect lightweight choice.

## Constraints
* Ensure that you do not define a `table:` attribute in any `.spy.yaml` files when working with Serverpod mini.
