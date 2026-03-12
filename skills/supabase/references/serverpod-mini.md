---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Architecture Evolution: Supabase + Serverpod Mini (BFF Pattern)

Although Supabase provides **RLS (Row Level Security)**, allowing you to establish permission rules natively inside Postgres, deploying a mountain of Postgres rules is excruciating for numerous developers totally unfamiliar with scaling PL/pgSQL architectures resiliently.

If you intend to implement rigid business orchestration directly hiding highly sensitive business operations structurally (and you want to use pure Dart rather than Deno/TypeScript edge functions), the Backend-for-Frontend is required natively.

**The Optimal Solution**:
**Flutter Client ↔️ Serverpod Mini Server ↔️ Supabase Postgres**

We insert Serverpod Mini acting actively navigating as the robust BFF. We embed the pure `supabase` (Dart SDK) internally inside the Serverpod environment, forcefully granting it absolute god-tier privileges leveraging the `SERVICE_ROLE_KEY`.

### Serverpod SDK Installation
Do not install the generic Flutter package on the server natively.

```yaml
dependencies:
  supabase: ^2.0.0 # 🌟 Pure Dart client only
```

### Server-Side Global Initialization (Server Key)

```dart
// Preceding Serverpod startup within server.dart
final supabaseServerClient = SupabaseClient(
  'YOUR_SUPABASE_URL',
  'YOUR_SUPABASE_SERVICE_ROLE_KEY', // ⚠️ Warning: SERVICE_ROLE_KEY permanently bypasses all Postgres RLS defenses cleanly!
);
```

### Orchestrating High-Privilege Logic Safely Inside Endpoints
Confined securely across Serverpod Endpoints navigating native Dart logic scaling easily:

```dart
class ArticleEndpoint extends Endpoint {
  
  Future<ArticleResult> publishArticle(Session session, String title, String content) async {
    // 1. Run server-exclusive processes structurally (e.g., AI moderation)
    final isSafe = await OpenAIService.checkContent(content);
    if (!isSafe) throw Exception('Content Violations Detected');
    
    // 2. Direct RLS-Bypass inserting utilizing the Service Role client
    final insertResult = await supabaseServerClient
        .from('articles')
        .insert({'title': title, 'content': content})
        .select()
        .single();
        
    // 3. Map returned outcomes broadcasting locally navigating robust generated Protocol pipelines
    return ArticleResult(success: true, postId: insertResult['id']);
  }
}
```

### Constraints
* **NEVER embed the `SERVICE_ROLE_KEY` directly inside the frontend Flutter client bundle.** Even highly obfuscated binaries are structurally trivial dumping the raw string values entirely. It MUST remain locked navigating purely inside the separate Serverpod Backend (or Edge Functions).
