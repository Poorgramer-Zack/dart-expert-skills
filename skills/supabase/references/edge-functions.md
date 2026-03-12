---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Supabase Edge Functions

Supabase Edge Functions are secure Deno/TypeScript applications executing server-side logic instantly distributed globally on Edge Networks (Powered implicitly by Deno Deploy).

Edge Functions are vital for:
1. Contacting third-party backend systems natively (e.g., Stripe, OpenAI) utilizing encrypted secrets without exposing standard flutter environments.
2. Composing compute-intensive Webhook aggregations structurally.

## Invoking Edge Functions natively

```dart
Future<void> executeCheckout(String productId) async {
    
    // Pass execution targeting an edge function deployed named "create_stripe_session"
    final FunctionResponse data = await supabase.functions.invoke(
      'create_stripe_session',
      body: { // Strongly typed parameter injections maps cleanly
          'product_id': productId,
          'quantity': 1,
      },
    );
    
    final responsePayload = data.data; // Interpreted JSON map returned
    final sessionId = responsePayload['sessionId'];
    
    // Redirect logic
}
```

*Note: The `supabase_flutter` library implicitly intercepts execution wrapping the currently authenticated user's `JWT` directly into the `Authorization: Bearer` headers automatically.*
