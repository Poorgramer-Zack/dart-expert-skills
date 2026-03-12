---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# 08 - Serverpod Mini (BFF Architecture)

## Goal
Completely bypass the limitations of Firestore Security Rules and safely integrate third-party APIs (Stripe, OpenAI) by routing the Flutter app's Firebase interactions through a highly secure Backend-For-Frontend (BFF) built in Dart using Serverpod Mini.

## The Architecture
Instead of using the standard Firebase Client SDKs to interact with Firestore, the Flutter app only communicates with Serverpod generated Endpoints. 

Serverpod runs on a cloud server (like Google Cloud Run), initializes the `dart_firebase_admin` SDK, and wields absolute administrative authority to read/write to Firestore.

### Why do this?
1. **Ditch Security Rules:** Authoring complex authorization logic in `.rules` files is extremely error-prone. With this architecture, you write all authentication checks in standard, debuggable Dart code on the Serverpod backend.
2. **Hide API Keys:** If you need to charge a user via Stripe when they create a document, putting the Stripe Secret Key in the Flutter app is an instant security breach. In this architecture, the Secret Key lives safely on the Serverpod backend.
3. **End-to-End Type Safety:** Data traversing from Client to Server is cryptographically armored by Serverpod YAML protocol files.

## Instructions

### 1. Serverpod Backend Initialization (`myminipod_server`)
Add the `dart_firebase_admin` package to your Serverpod backend dependencies. Initialize it during server startup.

```dart
// myminipod_server/lib/server.dart
import 'package:dart_firebase_admin/dart_firebase_admin.dart';

void run(List<String> args) async {
  // 1. Initialize the Admin App using a Service Account Key
  // (Store the JSON contents in Serverpod's Passwords.yaml or ENV vars)
  final admin = FirebaseAdminApp.initializeApp(
    'your-project-id',
    Credential.fromServiceAccountParams(
      clientId: '...',
      email: '...',
      privateKey: '...',
    ),
  );
  
  // 2. Instantiate Admin services
  final firestore = Firestore(admin);
  
  // 3. Inject them into Serverpod's context globally or let endpoints construct them
  final pod = Serverpod(args, ...);
  
  // Example: Store it globally so endpoints can grab it
  // pod.customData['firestore'] = firestore; 
  
  await pod.start();
}
```

### 2. Creating the Secure Endpoint
In the backend, create an endpoint that handles complex business logic (like processing a payment AND updating Firestore simultaneously).

```dart
// myminipod_server/lib/src/endpoints/user_transaction_endpoint.dart
class UserTransactionEndpoint extends Endpoint {
  
  // This endpoint is called from the Flutter Client
  Future<TransactionResult> processPurchase(Session session, String userId, double amount) async {
    // 1. You are running on a secure server. Nobody can decompile this code.
    
    // 2. Interact with Third-Party Payment API
    final paymentSuccess = await StripeService.chargeCard(userId, amount);
    
    if (!paymentSuccess) {
      return TransactionResult(success: false); // Defined in a .yaml protocol
    }

    // 3. Write securely to Firestore using ADMIN privileges. No .rules apply here!
    // Initialize Firestore admin (or grab it from session.server.customData)
    final firestore = Firestore(adminApp);
    
    await firestore.collection('receipts').add({
      'userId': userId,
      'amount': amount,
      'timestamp': DateTime.now().toIso8601String(),
    });
    
    // 4. Return strongly-typed success to Flutter
    var result = TransactionResult(success: true, receiptId: "...");
    return result;
  }
}
```

## Constraints
*   **Dual Dependency Awareness:** The Flutter Client will STILL use the `firebase_auth` Client SDK to authenticate the user and obtain a JWT token. The Client then passes this JWT Token to the Serverpod Endpoint, which uses `dart_firebase_admin` to verify that the token is legitimate before executing backend code.
*   **Admin Power:** The Admin SDK ignores ALL Firestore Security Rules. Ensure your Dart logic inside the Serverpod Endpoint rigorously checks if the user making the request actually has the authority to modify the targeted documents.
