---
name: "auto-route-best-practices"
description: "Best practices and guidelines for AutoRoute Latest Version (v11.1.x). Use this skill when you need to write or review code related to AutoRoute Latest Version (v11.1.x)."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# AutoRoute Latest Version Best Practices Guide (v11.1.x)

## Goal
`auto_route` is a powerful, compile-time Code generation based navigation package for Flutter. Compared to GoRouter requiring hand-written nested definitions, AutoRoute provides a purely minimalist development experience via Annotations (`@RoutePage()`). The current latest version is roughly **11.1.0**.

⚠️ **Important Reminder**: AutoRoute introduced major Breaking Changes in v11.0.0. Many legacy tutorial syntaxes are completely invalid.

## Instructions

### 1. Core Concept and Dependency Installation
By scanning `@RoutePage()` annotations across pages, AutoRoute automatically generates navigation code encapsulating strongly-typed parameters, making jumping and parameter passing entirely automated and type-safe.

#### Dependency Setup
```yaml
dependencies:
  auto_route: ^11.1.0

dev_dependencies:
  auto_route_generator: ^11.1.0
  build_runner: ^2.4.0
```

### 2. Page Definition and Parameter Passing (Best Practices)
Say goodbye to handwritten parsers! This is AutoRoute's greatest strength: the native constructor *is* your parameter.

```dart
import 'package:auto_route/auto_route.dart';
import 'package:flutter/material.dart';

// ✅ v11+ no longer requires writing return types. E.g., @RoutePage<bool>() is deprecated
@RoutePage() 
class ProductDetailsScreen extends StatelessWidget {
  // Automatic strong type constraints
  final int productId;
  final String productName;

  const ProductDetailsScreen({
    super.key,
    required this.productId,
    required this.productName,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(productName)),
      body: Center(child: Text('Product ID: $productId')),
    );
  }
}
```

### 3. Global Route Definition (AppRouter) 🚀 v11 Major Update Syntax
In version 11, the Router class no longer extends `$AppRouter`; rather, it inherits from `RootStackRouter` within the package.

```dart
import 'package:auto_route/auto_route.dart';
// The part file MUST be declared, otherwise code generation will fail
part 'app_router.gr.dart'; 

@AutoRouterConfig(replaceInRouteName: 'Screen|Page,Route') 
class AppRouter extends RootStackRouter { // 🌟 v11 MUST inherit RootStackRouter
  
  // Override the routes array providing the route tree
  @override
  List<AutoRoute> get routes => [
    // Root directory (Initial page)
    AutoRoute(page: HomeRoute.page, initial: true),
    
    // Other page declarations
    AutoRoute(page: ProductDetailsRoute.page),
  ];
}
```
*💡 After writing, execute `dart run build_runner build -d`.*

#### main.dart Initialization Setup
```dart
void main() {
  // Create Router instance, ideally declared globally or provided via dependency injection
  final appRouter = AppRouter();
  
  runApp(MaterialApp.router(
    routerConfig: appRouter.config(), // Hook up AutoRoute configuration
  ));
}
```

### 4. Navigation Operations on UI Best Practices
Via the compiled `ProductDetailsRoute`, typos in variable names are completely prevented. Additionally, v11 removed many legacy APIs (e.g., `pushNamed`, `replaceNamed`).

```dart
class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      child: const Text('Go to Product'),
      onPressed: () async {
        // ✅ Best Practice: Use the auto-generated Route object to execute strongly-typed jumps
        context.router.push(ProductDetailsRoute(
          productId: 123, 
          productName: 'iPhone 15'
        ));

        // 🌟 v11 To await a return result, specify the generic directly on pushRoute:
        // final result = await context.pushRoute<bool>(MyDialogRoute());

        // 🌟 v11 API Changes:
        // context.router.navigatePath('/path') (Replaces legacy navigateNamed)
        // context.router.replacePath('/path')  (Replaces legacy replaceNamed)
        // context.router.pop()                 (Replaces legacy popForced)
      },
    );
  }
}
```

### 5. Nested Routing (Nested Routing / Bottom Navigation Bar)
Use `AutoTabsRouter` to manage internal stacks automatically.

#### Route Configuration
```dart
@override
List<AutoRoute> get routes => [
  AutoRoute(
    page: DashboardRoute.page, 
    initial: true,
    children: [
      AutoRoute(page: ProfileTabRoute.page),
      AutoRoute(page: SettingsTabRoute.page),
    ]
  ),
];
```

#### Dashboard Page Implementation
Stop manually coding `IndexedStack`; let `AutoTabsRouter` automatically manage internal stacks.

```dart
@RoutePage()
class DashboardScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return AutoTabsRouter(
      // Configure underlying tab mapping configurations
      routes: const [ProfileTabRoute(), SettingsTabRoute()],
      builder: (context, child) { // v11 removed the animation parameter; use transitionBuilder for animations
        // Extract current info from tabsRouter
        final tabsRouter = AutoTabsRouter.of(context);
        
        return Scaffold(
          body: child, // This block will automatically toggle displaying child pages
          bottomNavigationBar: BottomNavigationBar(
            currentIndex: tabsRouter.activeIndex,
            onTap: tabsRouter.setActiveIndex, // Perfectly toggles the internal route tree
            items: const [
              BottomNavigationBarItem(label: 'Profile', icon: Icon(Icons.person)),
              BottomNavigationBarItem(label: 'Settings', icon: Icon(Icons.settings)),
            ],
          ),
        );
      },
    );
  }
}
```

### 6. Route Guards (Guards)
The optimal solution for handling authentication is implementing an `AutoRouteGuard`.

```dart
class AuthGuard extends AutoRouteGuard {
  @override
  void onNavigation(NavigationResolver resolver, StackRouter router) {
    final isAuthenticated = checkAuthToken(); 
    
    if (isAuthenticated) {
      resolver.next(true); // Authorize passage
    } else {
      // Redirect to the login page; trigger resolver.next(true) upon successful login
      router.push(LoginRoute(onResult: (success) {
        resolver.next(success);
      }));
    }
  }
}
```
*   **Binding a Guard**: In config: `AutoRoute(page: ProfileRoute.page, guards: [AuthGuard()])`
*   **v11 Global Guards**: v11 supports a global guards list: `final appRouter = AppRouter(guards: [AuthGuard()]);`

### 7. Summary
1.  **Type safety without effort**: Retains the feeling of writing regular Widget code, yet yields all necessary jump protections behind the scenes.
2.  **v11 Architectural Upgrade**: Fully transitioned to utilizing `RootStackRouter` inheritance, abandoned legacy `$Router` prefix extension, and updated all String path jumping APIs (`navigatePath` etc.).
3.  **Micro-Frontend Dependency (Self-contained capability)**: `PageRouteInfo` in v11 has become a completely self-sufficient object, which is highly advantageous for route integration in multi-package/modular development.

## Constraints
* Always remember to run `dart run build_runner build -d` after modifying or adding `@RoutePage()` annotations or the `AppRouter`.
* Do NOT use legacy `v10` string path methods or `.gr.dart` `$Router` extensions when working in `v11` projects.
