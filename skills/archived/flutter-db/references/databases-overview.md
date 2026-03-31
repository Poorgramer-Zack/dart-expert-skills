---
name: "flutter-databases"
description: "Work with databases in a Flutter app"
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# flutter-data-layer-persistence

## Goal
Architects and implements a robust data layer in Flutter applications. Establishes a single source of truth using the Repository pattern, isolates external API and local database interactions into stateless Services, and implements optimal local caching strategies based on data requirements.

## Decision Logic
Evaluate the user's data persistence requirements using the following decision tree to select the appropriate caching strategy:

*   **Is the data small, simple key-value pairs (e.g., user settings, theme)?**
    *   *Yes:* Use `shared_preferences` (via static wrapper).
*   **Is the data sensitive (e.g., OAuth tokens, passwords, private keys)?**
    *   *Yes:* Use `flutter_secure_storage`.
*   **Is the data a large, structured, relational dataset requiring complex queries?**
    *   *Yes:* Use `drift` (Reactive SQL).
*   **Is the data a large, high-performance unstructured/NoSQL dataset?**
    *   *Yes:* Use `hive_ce`.
*   **Is the data primarily images or large files?**
    *   *Yes:* Use `path_provider` + `cached_network_image`.

## Instructions

1. **Analyze Data Requirements**
   **STOP AND ASK THE USER:** "What specific data entities need to be managed in the data layer, and what are their persistence requirements (e.g., sensitivity, relational complexity, performance needs)?"
   *Wait for the user's response before proceeding.*

2. **Configure Dependencies**
   Update `pubspec.yaml` based on the chosen technology from the Decision Logic.

3. **Establish Data Models**
   Define clean Domain models. If using `drift` or `hive_ce`, ensure generated code is prepared.

4. **Implement Stateless Services**
   Create pure wrappers for the chosen database (e.g., `Prefs` static class for shared\_prefs, `Database` class for Drift). Services should be stateless from the application's perspective.

5. **Implement the Repository**
   The Repository acts as the single source of truth. It manages syncing between the Local Service and Remote API.
   *   *Rule:* UI components must NEVER talk to Services directly.

6. **Validate Implementation**
   *   *Check:* Is sensitive data encrypted in `flutter_secure_storage`?
   *   *Check:* Does the repository handle database opening/initialization correctly?
   *   *Check:* Are all calls to `shared_preferences` encapsulated in a static helper?

## Constraints
*   **Single Source of Truth:** All data requests must route through the Repository; never bypass it via direct Service calls.
*   **Security First:** Strictly prohibit storing tokens or PII in `shared_preferences` or unencrypted `hive`.
*   **Statelessness:** Services should handle I/O and not maintain application logic state.
*   **Type Safety:** Avoid `dynamic` or raw JSON handling outside the data layer. Convert to models early.
