---
name: "flutter-animate"
description: "Declarative animation patterns using the flutter_animate package. Use when adding UI animations, transitions, staggered effects, or fixing animation performance (jank)."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Flutter Animate Implementation

## Goal
Implement highly performant, composable, and declarative animations in Flutter using the `flutter_animate` package. The goal is to aggressively reduce boilerplate by utilizing `.animate()` extension methods on Widgets rather than manually orchestrating `AnimationController` and `StatefulWidget` mechanics.

## Instructions

### 1. The Core Extension Syntax
Instead of wrapping widgets in an explicit `Animate` builder block, universally prefer the `.animate()` extension method on any Widget followed by chained effects.

**Correct Usage:**
```dart
// 🌟 Best Practice: Extension chaining
Text("Hello World!")
  .animate()
  .fade(duration: 500.ms)
  .scale(curve: Curves.easeOutBack);
```

**Avoid (unless strictly necessary):**
```dart
// ❌ Avoid explicit nesting unless building a highly complex reusable list
Animate(
  effects: [FadeEffect(), ScaleEffect()],
  child: Text("Hello World!"),
)
```

### 2. Time Extensions (num)
Always utilize the provided `num` extensions for `Duration` assignments. This dramatically improves declarative readability.
*   `500.ms` (Milliseconds)
*   `2.seconds` (Seconds)
*   `1.5.minutes` (Minutes)

### 3. Orchestration & Sequencing (`.then()`)
When effects need to run sequentially rather than in parallel, use the `.then()` pseudo-effect. It establishes a new baseline time equal to the previous effect's completion time.

```dart
Column(
  children: [
    Text("Step 1").animate().fadeIn(duration: 400.ms).slideX(),
    Text("Step 2").animate()
      .fadeIn(duration: 400.ms)
      .then(delay: 200.ms) // Step 2 slides 200ms AFTER its own fade completes
      .slideY(),
  ],
)
```

### 4. Reactive State Animations (`target`)
The most potent feature of `flutter_animate`. You can completely eliminate `AnimatedContainer` or `AnimatedOpacity` by using the `target` property. 
When `target: 1`, the animation rests at its `end` values. When `target: 0`, the animation reverses cleanly to its `begin` values.

```dart
// Inside a build method that listens to State or Riverpod
bool isHovered = ref.watch(hoverStateProvider);

MyButton()
  .animate(target: isHovered ? 1 : 0) // 🌟 Drives the animation entirely by State
  .fade(end: 0.8)
  .scaleXY(end: 1.1, curve: Curves.easeOutExpo);
```

### 5. Infinite Loops & Callbacks
To repeat an animation indefinitely, utilize the `onPlay` callback to manipulate the internal controller.

```dart
Icon(Icons.warning)
  .animate(onPlay: (controller) => controller.repeat(reverse: true))
  .fadeOut(curve: Curves.easeInOut);
```

### 6. Value Listeners & Custom Effects
If you need to extract the raw `0.0` to `1.0` double value (e.g. to drive a custom painter or a complex color lerp), use `.listen()` or `.custom()`.

**Listen (Side-Effects):**
```dart
Text("Hello").animate().fadeIn(duration: 1.seconds)
  .listen(callback: (value) => print('Current alpha: $value'));
```

**CustomEffect (Building from interpolation):**
```dart
Text("Color Shift").animate().custom(
  duration: 300.ms,
  builder: (context, value, child) => Container(
    // Extrapolates colors cleanly entirely within the animation pipeline
    color: Color.lerp(Colors.red, Colors.blue, value),
    child: child, 
  )
);
```

### 7. Logical Toggles & UI Swapping
*   `ToggleEffect`: Yields a `bool` based on whether the animation has finished its duration. Useful for flipping raw properties (like `AbsorbPointer` locks).
*   `SwapEffect`: Replaces the entire rendered Widget at the very end of the animation timeline.

```dart
// The button fades out, and is instantly replaced by a Loader widget when the fade completes.
SubmitButton().animate()
  .fadeOut(duration: 300.ms)
  .swap(builder: (context, child) => const CircularProgressIndicator());
```

## Constraints
*   **Extension Priority**: Always default to the `.animate()` extension chaining syntax for generating animations unless directly generating an `AnimateList`.
*   **State Simplicity**: Do not define `AnimationController` inside a `StatefulWidget` if the goal can be solely achieved using `flutter_animate`'s `target:` or `onPlay:` properties.
*   **Reuse Effects**: If building a Design System, extract `List<Effect>` arrays into global constants to guarantee application-wide transition consistency.
