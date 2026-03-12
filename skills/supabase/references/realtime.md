---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Supabase Realtime & Presence

Supabase leverages Postgres' native replication capabilities (CDC) to provide real-time stream updates.

## Streaming Data (StreamBuilders)
The absolute best practice for dynamic UI mapping is directly translating Supabase developments natively inside a `StreamBuilder`. 

```dart
Stream<List<Message>> getMessageStream(int roomId) {
  // .stream() requires a primary key parameter to perform efficient row-diffing underlying
  return supabase
      .from('messages')
      .stream(primaryKey: ['id'])
      .eq('room_id', roomId)
      .order('created_at')
      .map((maps) => maps.map((map) => Message.fromJson(map)).toList());
}
```

### ⚠️ CRITICAL Realtime Constraints
**Always remember to cancel the subscription (`.cancel()`) or explicitly wrap instances inside disposing architecture patterns (like Riverpod's automatic lifecycle disposal) when navigating away.** Unlike standard static Futures, orphaned Websocket streams continuously pull traffic endlessly causing catastrophic background memory leaks on mobile platforms natively.

## Presence ("I'm Online" Features)
Presence lets you easily detect if peers are online, typing, or navigating elsewhere without constantly polling the main Database payload.

```dart
// 1. Instantiate the channel specifically dedicated universally
final myChannel = supabase.channel('room_1');

// 2. Map listener events
myChannel
  .onPresence(
    event: PresenceEvent.sync,
    callback: (payload) {
        final onlineUsers = myChannel.presenceState();
        // Update your UI utilizing the synchronized mapping
    }
  )
  .onPresence(
    event: PresenceEvent.join,
    callback: (payload) {
        // Users popped online
    }
  )
  .onPresence(
    event: PresenceEvent.leave,
    callback: (payload) {
        // Users abruptly detached
    }
  )
  .subscribe((status, [error]) async {
      // 3. Inform the server of YOUR state AFTER subscribing smoothly
      if (status == RealtimeSubscribeStatus.subscribed) {
          await myChannel.track({
              'online_at': DateTime.now().toIso8601String(),
              'status': 'typing'
          });
      }
  });

// 4. Detach flawlessly
await supabase.removeChannel(myChannel);
```
