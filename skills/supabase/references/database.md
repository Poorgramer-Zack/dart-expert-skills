---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Supabase Postgres Database

Because Supabase is powered purely by Postgres, it abandons NoSQL (Collections/Documents) structuring for true Relational logic (Tables/Rows).

## Strongly Typed & JSON Mapping
By default, `.select()` operations yield untyped JSON arrays: `List<Map<String, dynamic>>`. Just like Firebase `.withConverter`, the absolute best practice dictates running all queries cleanly through `Freezed` `.fromJson()` mappings instantly.

### Basic Fetch
```dart
Future<List<Article>> fetchArticles() async {
  final response = await supabase
      .from('articles')
      .select()
      .eq('published', true) // Filter
      .order('created_at', ascending: false) // Order
      .limit(10); // Paginate
      
  return (response as List).map((row) => Article.fromJson(row)).toList();
}
```

### Powerful Outer/Inner Joins Natively
Because of PostgREST (the API backend driving Supabase), you can request related Foreign Key data synchronously via a GraphQL-styled string notation:

```dart
Future<List<Comment>> fetchUserComments(String userId) async {
  final response = await supabase
      .from('comments')
      // Pull all columns from comments, PLUS the 'username' column from the 'users' table
      .select('*, users!inner(username)') 
      .eq('user_id', userId);
      
  return (response as List).map((row) => Comment.fromJson(row)).toList();
}
```
*Note the `!inner` clause. By default, joins act as `LEFT JOIN`. Adding `!inner` forces an `INNER JOIN`, filtering out rows where the foreign relation might be entirely missing.*

## Modifications (Insert / Update / Delete)

```dart
// Insert a new row
await supabase.from('users').insert({
  'username': 'FlutterMaster',
  'age': 25,
});

// Update rows
await supabase.from('users').update({
  'age': 26,
}).eq('username', 'FlutterMaster');

// Delete rows
await supabase.from('users').delete().eq('age', 26);
```

## Handling Return Execution Values
Often, upon inserting or updating, you immediately require the Postgres-generated UUID or `created_at` timestamp. Chain `.select().single()` instantly onto the mutation to retrieve it in precisely one network roundtrip.

```dart
final newlyCreatedMap = await supabase
    .from('articles')
    .insert({'title': 'Supabase rules'})
    .select() // Ask Postgres to return the generated payload
    .single(); // Enforce map extraction instead of list

final article = Article.fromJson(newlyCreatedMap);
```
