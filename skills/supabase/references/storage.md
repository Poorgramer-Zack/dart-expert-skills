---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Supabase Storage

Supabase establishes "Buckets" containing Files structurally, operating highly similarly to Firebase Cloud Storage, utilizing RLS Security Policies for profound authorization boundaries natively.

## Mobile Native File Uploads

```dart
import 'dart:io';

Future<String> uploadAvatar(File imageFile, String userId) async {
  final fileExtension = imageFile.path.split('.').last;
  
  // Construct a unique path (userId/timestamp.jpg)
  final fileName = '${DateTime.now().millisecondsSinceEpoch}.$fileExtension';
  final storagePath = '$userId/$fileName';
  
  // 1. Perform execution explicitly targeting the "avatars" Bucket
  await supabase.storage.from('avatars').upload(storagePath, imageFile);
  
  // 2. Fetch the publicly readable URL (Assuming "avatars" bucket is Public)
  return supabase.storage.from('avatars').getPublicUrl(storagePath);
}
```

## Flutter Web File Uploads (`uploadBinary`)
On Flutter Web applications, `dart:io` `File` objects do not interact directly with native filesystems. Instead, file pickers generate generic byte arrays. 

To bridge this flawlessly mapping cross-platform, utilize the explicitly defined `uploadBinary` method on Web targets.

```dart
import 'dart:typed_data';

Future<String> uploadWebAvatar(Uint8List imageBytes, String storagePath) async {
    // ⚠️ Mandatory 'uploadBinary' execution for byte stream payloads natively
    await supabase.storage.from('avatars').uploadBinary(storagePath, imageBytes);
    
    return supabase.storage.from('avatars').getPublicUrl(storagePath);
}
```
