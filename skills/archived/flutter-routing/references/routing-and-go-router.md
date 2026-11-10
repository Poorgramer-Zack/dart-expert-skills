---
name: "flutter-routing-and-navigation"
description: "Comprehensive guide for implementing navigation and deep-linking utilizing GoRouter natively"
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---
# GoRouter & Navigation Implementations (v14.x)

## Goal
GoRouter spearheads the official Flutter ecosystem's endorsed Declarative Routing solution. This definitive guide completely replaces the legacy imperative `Navigator.push` paradigms, enforcing scalable **Declarative Navigation**, unified path manipulation, robust authentication redirections, and persistent bottom-navigation architectures natively.

## Instructions

### 1. Installation 
```yaml
dependencies:
  go_router: ^14.0.0

dev_dependencies:
  build_runner: ^2.4.0
  go_router_builder: ^4.0.0
```

### 2. Basic Routing Setup Configurations
Initialize the `GoRouter` instance directly, deploying raw Dart `GoRoute` components forming the global application routing web cleanly.

```dart
import 'package:go_router/go_router.dart';

final router = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      name: 'home', // 🌟 Named route drastically assists precise context.goNamed access natively
      path: '/',
      builder: (context, state) => const HomeScreen(),
      routes: [
        // This signifies a nested child route (Nested Route) tied beneath home.
        // NOTE: A nested route's path categorically CANNOT consist of a preceding '/'.
        GoRoute(
          name: 'detail',
          path: 'detail/:id', // Incorporating Path Parameter explicit declarations
          builder: (context, state) {
            // Extracting URL variables straight out of state.pathParameters
            final id = state.pathParameters['id']!;
            // Deconstructing Query parameters strictly from state.uri.queryParameters (e.g. ?sort=desc)
            final sort = state.uri.queryParameters['sort'];
            
            return DetailScreen(id: id, sort: sort);
          },
        ),
      ]
    ),
  ],
);
```

### 3. Execution Jumps and Navigations

It is passionately recommended to absolutely distinctuate the contrast between `.go` and `.push`:
*   **`.go()` (Primary Core Base)**: Immediately alters the underlying URL address format, abandoning the incumbent stack layout dynamically switching trees entirely toward the brand-new designated route. Excellently suited navigating bottom-tab logic mechanisms or root homepage transitions uniformly seamlessly.
*   **`.push()`**: Thoughtlessly overlays the newest generated page atop the prevalent visible screen blindly (carrying its inherent back button out of the box). This behavior closely orchestrates standard legacy Navigator actions but **categorically corrupts Web URL native back-page behavior experiences**. Instructively designated utilizing solely for ephemeral fleeting superimposed display views manually natively.

```dart
// 1. Traversal deploying explicit Route Names (Highly best practice! Diminishes typographical URL slash disasters)
context.goNamed(
  'detail',
  pathParameters: {'id': '123'},
  queryParameters: {'sort': 'desc'},
);

// 2. Traverse jumping implementing absolute full-length URLs (Location)
context.go('/detail/123?sort=desc');

// 3. Bruteforce superimposition Stack Overlays (Push)
context.push('/detail/123'); 
```

### 4. Bottom Navigation Bar Ecosystem Integrations (`StatefulShellRoute`)

Addressing scenarios demanding "Persistent bottom navigation tabs maintaining independent scroll states without triggering re-render cascades upon toggling", **violently discard legacy habits of hand-coding IndexedStack layers**. Transition universally towards implementing `StatefulShellRoute` resolving compartmentalized scroll cache positions seamlessly protecting all disparate separated nested routing layer stacks elegantly robustly natively!

```dart
final router = GoRouter(
  initialLocation: '/home',
  routes: [
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        // Formulate returning your customized underlying Scaffold (Embedding the user-defined BottomNavigationBar directly here)
        return MainShellScaffold(navigationShell: navigationShell);
      },
      branches: [
        // Tab Branch 1: Homepage Base
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/home',
              builder: (context, state) => const HomeScreen(),
            ),
          ],
        ),
        // Tab Branch 2: Setting Control Panel Hub
        StatefulShellBranch(
          routes: [
             GoRoute(
              path: '/settings',
              builder: (context, state) => const SettingsScreen(),
            ),
          ],
        ),
      ],
    ),
  ],
);

// -------------------------------------------------------------
// Inside your MainShellScaffold (possessing its isolated generic BottomNavigationBar):
// Vigorously trigger exactly `navigationShell.goBranch` finalizing pure Tab swaps
BottomNavigationBar(
  currentIndex: navigationShell.currentIndex,
  onTap: (index) => navigationShell.goBranch(
    index,
    // Employs intrinsic conditional triggers autonomously clearing inner-stack layers routing immediately backwards returning initial Base Page when user re-clicks the identical currently focused Tab.
    initialLocation: index == navigationShell.currentIndex, 
  ),
  items: const [...],
)
```

### 5. Unified Route Guards and Redirections (Auth Guard)

Strategizing access validations enforcing logins categorically MUST avoid blocking operational screens at trivial Widget layers. Consolidate every protective barrier within standard `GoRouter`'s innate `redirect` structural controls.

```dart
final router = GoRouter(
  // ...
  redirect: (context, state) {
    // Acquire centralized authentication operational states exclusively
    final isLoggedIn = context.read<AuthBloc>().state.isAuthenticated; 
    final isGoingToLogin = state.matchedLocation == '/login';
    
    if (!isLoggedIn && !isGoingToLogin) {
      // Intruders lacking authorizations probing elsewhere -> Forcefully kick bounds routing toward the Login logic natively.
      return '/login';
    }
    
    if (isLoggedIn && isGoingToLogin) {
      // Authenticated users absurdly attempting visiting Login -> Bounced kicking returning actively towards Homepage hub natively.
      return '/home';
    }
    
    return null; // Demanding NO interceptions required, explicitly permitting unimpeded passage progression natively!
  },
);
```

### 6. Strongly-Typed Routing Framework (`go_router_builder`)

To eradicate fragile runtime URL string-matching vulnerabilities definitively, deploy `go_router_builder` architecting robust, compile-time verified routing mechanisms natively.

**1. Route Declarations:**
Define routes extending `GoRouteData` aggressively utilizing `@TypedGoRoute` annotations forming hierarchical structures.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

// Imperatively required part directive
part 'router.g.dart';

@TypedGoRoute<HomeRoute>(
  path: '/home',
  routes: [
    TypedGoRoute<DetailRoute>(path: 'detail/:id'),
  ],
)
class HomeRoute extends GoRouteData {
  const HomeRoute();
  
  @override
  Widget build(BuildContext context, GoRouterState state) => const HomeScreen();
}

class DetailRoute extends GoRouteData {
  const DetailRoute({required this.id, this.sort});
  
  final String id; // Represents Path Parameter natively
  final String? sort; // Represents Query Parameter natively
  
  @override
  Widget build(BuildContext context, GoRouterState state) => DetailScreen(id: id, sort: sort);
}
```

**2. Router Initialization:**
The builder uniquely generates a unified `$appRoutes` resolving all top-level instantiated routes collectively.

```dart
final router = GoRouter(
  initialLocation: '/home',
  routes: $appRoutes, // Automatically aggregated global map
);
```

**3. Type-Safe Navigation:**
Execute transitions flawlessly leveraging instance-bound `.go(context)` methods exclusively. This secures parameter requirements validated completely at compilation-time, viciously blocking missing parameter runtime disasters!

```dart
// 🌟 Requirements strictly enforced explicitly at compile-time!
const DetailRoute(id: '123', sort: 'desc').go(context);
```

**4. Triggering Code Generation:**
Synthesize generated files systematically executing:

```bash
dart run build_runner build --delete-conflicting-outputs
```

## Constraints
* **No `Navigator.push` Paradigms:** Unceremoniously forsake deploying legacy `Navigator.push` exclusively. Radically shift operational contemplations exclusively prioritizing declarative URL-based `context.go` or `context.goNamed` maneuvers seamlessly!
* **Branch Slashes Strictness:** Absolutely avoid deploying `GoRouter` nested sub-routes initiating incorporating leading forward-slashes `/` within sub-route branches unconditionally.
* **Abolish `IndexedStack` Constructs:** Ensure replacing legacy explicit `IndexedStack` configurations meticulously relying wholly enforcing `StatefulShellRoute` branch structural dependencies exclusively securing reliable platform URL consistencies.
