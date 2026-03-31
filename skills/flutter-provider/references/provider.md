---
name: "provider-best-practices"
description: "Best practices and guidelines for Provider Latest Version Best Practices Guide (v6.1.x). Use this skill when you need to write or review code related to Provider Latest Version Best Practices Guide (v6.1.x)."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Provider Latest Version Best Practices Guide (v6.1.x)

## Goal
Implements state management and dependency injection using the `provider` package in Flutter applications. While the author of this package currently highly recommends migrating new projects to `Riverpod`, `provider` (currently around version **6.1.5**) still fundamentally underpins massive legacy environments worldwide. The goal is to enforce precise structural separations manipulating Model-View-ViewModel (MVVM) architectures and eliminate excessive unnecessary Widget rebuilds leveraging targeted state consumption.

## Instructions

### 1. Core Concept: Dependency Injection via ChangeNotifier
Provider's core philosophy is deeply rooted entirely upon Flutter's native `InheritedWidget` arrays. It efficiently transmits structural data downwards cascading across the tree topology, forcibly notifying dependent registered subscribed child Widgets invoking dynamic rebuilding occurrences wherever localized data sequences mutate.

#### Dependency Configuration
```yaml
dependencies:
  provider: ^6.1.5
```

### 2. Best Practice: Architecture & Global Provisioning
1.  **Strictly Provide Locally Resolving Minimal Node Hierarchies**: Absolutely avoid indiscriminately jamming every Provider at the zenith above the `MaterialApp` origin. If specific states universally solely dictate single isolated Page screens, strictly `Provide` them natively originating atop that specific targeted Page screen directly.
2.  **Consistently Deploy `MultiProvider` Layers**: Whenever accumulating arrays exceeding single Provider architectures, unconditionally deploy `MultiProvider` arrays circumventing horrific chaotic cascading nested callback doom structures globally.

```dart
// Native State Schema Paradigms
class CartModel extends ChangeNotifier {
  final List<Item> _items = [];
  List<Item> get items => _items;

  void add(Item item) {
    _items.add(item);
    notifyListeners(); // 🌟 Crucial mandate: Absolute failure invoking UI refreshes omitting this function.
  }
}

// Global Best Practice Injection Strategies
void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (context) => CartModel()),
        // 🌟 If this Model explicitly demands resolving secondary Providers proactively, formulate ProxyProvider:
        // ProxyProvider<CartModel, CheckoutModel>(...)
      ],
      child: const MyApp(),
    ),
  );
}
```

### 3. Best Practice: Consuming States Flawlessly
This marks the most devastatingly critical operational juncture impacting performance metrics intrinsically. Mandatory compliance is strictly enforced:

#### 3.1 Button Executions or Callback Logic: DEPLOY `.read()`
If you merely strictly require executing an isolated functional endpoint, **NEVER** instantiate reactive behavioral listeners actively.

```dart
// ✅ CORRECT: Execution proceeds without forcibly mutating rebuilding parent Button element topologies natively.
onPressed: () => context.read<CartModel>().add(item),

// ❌ DISASTROUS FATAL ERROR: Provokes absolute infinite rebuilding executions repeatedly crashing active arrays.
onPressed: () => Provider.of<CartModel>(context).add(item),
```

#### 3.2 Widget Architecture Rendering `build()` Methods: Minimize Listening Scopes Viscerally
**Exclusively champion deploying `Consumer` or `Selector` hierarchies**, completely prohibiting outright executing blunt direct top-level `context.watch()` engagements unconditionally.

```dart
// ❌ CATASTROPHIC ERROR: Forces absolute sweeping wholesale structural Scaffold rebuilds redundantly
Widget build(BuildContext context) {
  final cart = context.watch<CartModel>();
  return Scaffold(
    appBar: AppBar(title: Text('Cart')), // AppBar completely detached yet brutally suffers reconstruction overhead
    body: Text('Total: ${cart.items.length}'),
  );
}

// ✅ PERFECT EXECUTION: Precision targeting singularly exclusively rebuilding isolated explicit Text boundaries flawlessly
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(title: Text('Cart')), 
    body: Consumer<CartModel>(
      builder: (context, cart, child) {
        return Text('Total: ${cart.items.length}');
      },
    ),
  );
}
```

#### 3.3 Advanced Deep Minimization Methodologies: Utilizing `Selector` Dynamics
If wrestling massive colossal `UserModel` datasets harboring dozens of explicit parameters, yet the ongoing specific Widget exclusively strictly tracks the user's `name` string conditionally:

```dart
Selector<UserModel, String>(
  selector: (context, user) => user.name,
  builder: (context, name, child) {
    // 🌟 Exclusively dynamically rebuilds mapping selectively ONLY when UserModel `name` undergoes physical mutations identically natively!
    // Age or alternative nested parameters altering structurally definitively fails triggering reconstructive cascades altogether successfully!
    return Text('Hello, $name');
  },
);
```

### 4. Advanced Architectures: Constructing Complete MVVM Patterns
Deploying `provider` implementations throughout Flutter commands enforcing absolute mature MVVM (Model-View-ViewModel) abstractions universally unlocking impenetrable testable infrastructures reliably cleanly natively.

#### 4.1 Strict Responsibility Delineations
*   **Model Layer**: Dictates data array topologies (employing immutable `Freezed` generated architectures natively) communicating managing sophisticated remote Backend / Database schemas robustly (Repository / Service patterns). **Absolute prohibition referencing ANY `ChangeNotifier` or `BuildContext` arrays entirely here.** 
*   **ViewModel Layer**: Natively inheriting `ChangeNotifier` classifications. Universally accepts executing commands strictly generated by View layer logic executing downstream Model communications successfully managing returning data payloads exposing mutable properties publicly successfully cleanly selectively.
*   **View Layer**: Pure visual UI components exclusively actively strictly `watching` ViewModel property derivatives executing purely static UI rendering recalculations organically effortlessly naturally explicitly effortlessly identically safely!

#### 4.2 Code Execution Paradigm (Decoupled Flawlessly)

**ViewModel Construction:**
```dart
import 'package:flutter/material.dart';

// ViewModel rigorously inheriting ChangeNotifier, targeting precise localized Page-specific or functional domains
class LoginViewModel extends ChangeNotifier {
  final AuthRepository _repository; // Dependency injected Model repository seamlessly natively

  LoginViewModel(this._repository);

  bool _isLoading = false;
  bool get isLoading => _isLoading;

  String? _errorMessage;
  String? get errorMessage => _errorMessage;

  // Intentionally projected public Intents consumable directly via UI inputs universally
  Future<bool> login(String username, String password) async {
    _isLoading = true;
    _errorMessage = null;
    notifyListeners(); // Force loading states proactively globally 

    try {
      await _repository.authenticate(username, password);
      _isLoading = false;
      notifyListeners();
      return true; // Execution Success
    } catch (e) {
      _isLoading = false;
      _errorMessage = e.toString();
      notifyListeners(); // Project explicit explicit Error string diagnostics fundamentally 
      return false; // Execution Failure
    }
  }
}
```

**View Rendering Construct:**
```dart
class LoginScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // 1. Locally inject providing localized ViewModel structures explicitly targeting single Page bounds 
    return ChangeNotifierProvider(
      create: (context) => LoginViewModel(
         // Universally consume globally generated Repository components executing natively internally seamlessly 
         context.read<AuthRepository>(), 
      ),
      child: _LoginContent(), 
    );
  }
}

class _LoginContent extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Precision tracking specific variables mapping completely isolating Selector targets
    final isLoading = context.select<LoginViewModel, bool>((vm) => vm.isLoading);
    final errorMessage = context.select<LoginViewModel, String?>((vm) => vm.errorMessage);

    return Scaffold(
      body: Column(
        children: [
           if (errorMessage != null) Text(errorMessage, style: TextStyle(color: Colors.red)),
           ElevatedButton(
             onPressed: isLoading 
                 ? null 
                 : () async {
                     // 🌟 Executing pure read operations activating ViewModel underlying methodologies successfully implicitly effortlessly
                     final vm = context.read<LoginViewModel>();
                     final success = await vm.login('user', 'pass');
                     if (success) {
                        Navigator.pushReplacementNamed(context, '/home');
                     }
                 },
             child: isLoading ? CircularProgressIndicator() : Text('Login'),
           )
        ],
      )
    );
  }
}
```

## Constraints
* **Immutable Executions within `initState`**: Explicitly prohibited from unleashing `watch` directives inside `initState` configurations natively universally, generating catastrophic runtime exception anomalies entirely; utilize solely `context.read` implementations resolutely statically!
* **Prohibited Read Deployment within Widget Hierarchy Streams**: Forcibly banned unleashing `.read()` methods embedded executing traversing arbitrary Widget recursive tree generation flows `build(BuildContext context)` indiscriminately permanently universally destructively!
* **ChangeNotifier Abstraction**: Never integrate UI elements, rendering code, or context structures directly into the ViewModels (`ChangeNotifier`). Separation from UI code guarantees strict unit testing integrity seamlessly elegantly efficiently optimally naturally explicitly strictly reliably seamlessly!
