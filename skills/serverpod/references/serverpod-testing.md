---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Serverpod Testing Guide (Backend Framework Testing)

## Goal
Implements integration testing utilizing the `serverpod_test` framework. Because Serverpod features an ORM and its own Endpoint architecture, its testing solution provides a nearly perfect integration testing flow, allowing you to **seamlessly test your backend Endpoints and database interactions precisely as if calling local functions**.

## Instructions

### Project Configuration (Docker & YAML Settings)
Serverpod tests are **not Mocks**; they genuinely connect to an isolated **dedicated test database**.

#### 1.1 `docker-compose.yaml` (Ensure Test DB is Isolated from Production/Dev)
In the `myapp_server` directory, modify `docker-compose.yaml` to add dedicated Postgres and Redis services for testing (The port should avoid 8090/8091 which are used for standard development):

```yaml
services:
  # ... original postgres & redis environments
  
  # Add dedicated test database (uses port 9090)
  postgres_test:
    image: postgres:16.3
    ports:
      - '9090:5432'
    environment:
      POSTGRES_USER: postgres
      POSTGRES_DB: myapp_test
      POSTGRES_PASSWORD: "test_password_here"
    volumes:
      - myapp_test_data:/var/lib/postgresql/data
```

#### 1.2 `config/test.yaml` Parameters configuration
Create `test.yaml` beneath `config/` so Serverpod uses the corresponding port when generating the test server:
```yaml
# Test Server automatically finds available Ports (set to 0)
apiServer:
  port: 0
  publicHost: localhost
  publicPort: 0
  publicScheme: http

database:
  host: localhost
  port: 9090 # 🌟 Align with 9090 written in the docker-compose above
  name: myapp_test
  user: postgres
```
And make sure you provide the corresponding passwords in `config/passwords.yaml`.

#### 1.3 `config/generator.yaml`
Configure the path for automatically generated test tools:
```yaml
server_test_tools_path: test/integration/test_tools
```

### Auto-Generation & Package Installation
Add development dependencies in `pubspec.yaml`:
```yaml
dev_dependencies:
  # Version must EXACTLY MATCH the serverpod version your project uses!
  serverpod_test: ^3.4.0 
  test: ^1.24.2
```

**Execute the following after every Endpoint / Model YAML modification**:
```bash
serverpod generate
```
This command not only generates the Flutter Client but also generates `test/integration/test_tools/serverpod_test_tools.dart`, which is your core weapon for testing.

### Writing and Running Integration Tests

#### 3.1 Best Practice: use `withServerpod`
`withServerpod` helps you establish virtual states and Sessions, enabling you to invoke Endpoints directly!

```dart
import 'package:test/test.dart';
// 🌟 NEVER import from serverpod_test, import the generated test tools instead:
import 'test_tools/serverpod_test_tools.dart';

void main() {
  // withServerpod automatically sets up the test environment and connects to the test database
  withServerpod('Example Endpoint', (sessionBuilder, endpoints) {
    
    test('Calling hello() should return the correct greeting', () async {
      // 1. Act: Call the backend logic directly via endpoints!
      // sessionBuilder establishes state corresponding to this request's lifecycle
      final greeting = await endpoints.example.hello(sessionBuilder, 'Zack');
      
      // 2. Assert
      expect(greeting, 'Hello Zack');
    });

  });
}
```

#### 3.2 Pre-test Execution Setup
Since Serverpod tests connect to a real DB to guarantee reliability, the Test Docker Container must be active before execution:

```bash
cd myapp_server
# Launch Test and Development databases
docker compose up --build --detach
# Execute native dart testing commands
dart test
```

#### 3.3 Test Database Cleanup (Best Practice)
In more advanced tests, you might have write operations like `createUser`. `withServerpod` currently supports a **Rollback** mechanism:
Assuming your operations don't involve low-level edge cases, Serverpod's integration test system generally rolls back the DB Transaction automatically after each test completes, **ensuring tests remain independent and don't produce dirty data**. This is a highly critical and practical design feature!

## Constraints
* Never mock the database endpoints during integration tests within serverpod setups; rely on the real database configured in `docker-compose.yaml` to accurately test data persistence.
* Always launch the test container infrastructure via `docker-compose` prior to test execution to avoid connection refusals.
