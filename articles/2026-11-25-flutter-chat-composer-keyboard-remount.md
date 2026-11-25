---
layout: post
title: "The keyboard that vanished when the Flutter chat input expanded"
description: "Expanding a Flutter chat input dropped the keyboard and flashed. Two Flutter mechanics were at fault: a silent remount and a per-frame rebuild."
tags: showdev, flutter, ux, performance
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-25-flutter-chat-composer-keyboard-remount.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-25-flutter-chat-composer-keyboard-remount.png
---

You tap the expand button in a chat composer. The input flashes, grows to a few lines, and then the keyboard disappears. The screen is now a large, mostly empty editor, and the thing you were typing with is gone. On top of that, opening the keyboard in the first place feels heavier than it should — not broken, just a beat behind your thumb.

Both symptoms live in the same widget, but they have different causes. One is a widget-tree identity mistake. The other is a rebuild subscription that quietly turned the keyboard animation into a full-screen rebuild. This post walks through both, the fixes, and the two invariants that make a composer feel solid.

## Symptom one: the keyboard drops on expand

The composer had two layouts:

- Collapsed: a floating card with a frosted background, side margins and rounded corners.
- Expanded: a full-height editor that should reach from the header to the keyboard.

That sounds like a simple conditional. The code did it by swapping the widget returned from the composer wrapper: a glass card around the content while collapsed, and the bare content while expanded.

In Flutter, that is not a layout change. It is a **widget replacement**. The position in the tree stays the same, but the widget type at that position changes, so the framework cannot update the existing element. It unmounts the whole subtree and builds a new one.

For a composer, that subtree contains the `TextField`, which contains `EditableText`. When an `EditableText` is disposed, it closes its input connection. Closing the connection is what tells the platform input method to go away. In other words, the expand toggle was not resizing the keyboard-aware editor; it was tearing the editor down and building another one.

The symptoms follow from that one fact:

- The keyboard hides because its client disappeared.
- The field flashes because the new tree has to lay out from scratch.
- The expanded height looks wrong because the body grew while the keyboard was away, and then shrank again when the new field asked for focus.

## The first invariant: change siblings, not the element chain

The fix is to keep the composer content at the same position with the same widget type, and change only things around it. The frosting becomes a **sibling behind** the content inside a `Stack`, not a wrapper around it.

```dart
Stack(
  fit: StackFit.passthrough,
  children: [
    Positioned.fill(
      child: IgnorePointer(
        child: expanded ? editorSurface : glassBackdrop,
      ),
    ),
    content, // always the same position and type
  ],
)
```

The `Stack` gives the backdrop a stable slot. Expanding changes the backdrop widget, the frame padding, the radius and the surface color; the content itself is never reparented or replaced. The `TextField`, its focus node, its text state and the input connection all survive the toggle.

This is general Flutter advice, not a composer trick. Any time a mode switch changes the tree above a stateful leaf, ask whether the leaf is being *updated* or *replaced*. If the widget type at that slot changes, it is replaced. A `KeyedSubtree` with a global key can move an element, but the simpler and cheaper answer is usually to keep the structure constant and vary a sibling's properties.

Two details made the fix hold up:

- The expanded state also needed a visible surface. A translucent fill over a black chat background reads as empty space, so the expanded editor gets an opaque surface — except on the true-black theme, where it deliberately stays on the pure-black chat surface instead of becoming a gray slab.
- The test asserts element identity, not just pixel height: pump the composer, expand it, and verify the `TextField` element is the same object and still focused. A height assertion alone would have passed while the field was being recreated.

## Symptom two: the keyboard animation rebuilt the chat

The dropped keyboard was one bug. The sluggish keyboard was another.

A software keyboard does not appear in one jump. The platform reports a changing bottom inset over several frames, and Flutter resizes the Scaffold body to match. The question is what subscribes to that change.

The chat body was wrapped in a top-level `LayoutBuilder`. A `LayoutBuilder` rebuilds its subtree when its constraints change — and the keyboard changes the body's height on every animation frame. So every frame rebuilt the chat: the header, the whole message list and the composer.

There was a second subscription on top. The code decided whether the floating header should hide while the keyboard is up by reading `MediaQuery.of(context)` in `build`. `MediaQuery` is an inherited widget; depending on it means "rebuild me whenever anything in it changes", and the keyboard inset is part of that data. The route was also wrapped in a `MediaQuery` copy on every build. Same result: a rebuild per animation frame.

Two subscriptions, both aimed at the whole screen, for a change that only the geometry needs.

## The second invariant: a keyboard animation is a constraint animation

The fix was not to make `LayoutBuilder` faster. It was to stop asking the whole chat for a new layout on every frame.

- The body height is now reported by a small layout-reporting proxy after it lays out. The value goes into a notifier. Only the composer listens to it, because only the expanded editor needs the body height. While collapsed, the listener returns the same widget instance, so the animation does not rebuild the composer subtree at all.
- The "hide the header while the keyboard is up" decision is a boolean with a threshold. It is cached and recomputed from the view metrics only when the metrics change, and only a flip of the boolean schedules a build. The threshold crossing happens once per keyboard show, not once per frame.
- The route only reads the full `MediaQuery` while the status bar is deliberately pinned out of view. In the normal case, nothing in the chat build depends on the keyboard inset.

The result is that the keyboard animation still resizes the Scaffold body and still lays out the message viewport — that is unavoidable — but it no longer rebuilds the widgets. Layout remains; the per-frame rebuild work disappears.

| Before | After |
|---|---|
| Whole chat rebuilds every IME frame | Chat build is not scheduled by inset changes |
| Header-hide threshold read from `MediaQuery` in `build` | Cached boolean, recomputed on metrics changes |
| Message list rebuilt during the animation | List viewport lays out; widgets stay put |
| Composer rebuilds while collapsed | Composer returns the same child instance |

## The measurement that made it obvious

The most useful part of the debugging was not a profiler. It was the render tree from a debug build.

Dumping the render tree while the editor was expanded showed a composer card roughly 420 dp tall — exactly the space from the header to the keyboard. Inside it, the editable was 84 dp tall, sitting at the top. The card was filling the screen; the *editable* was not, and the contrast between the two was invisible on the true-black theme.

That measurement explained the "it only has a few lines" report without guessing. It also suggested the right visual fix: bottom-align the draft and the suffix actions next to the keyboard, and give the expanded surface an edge you can actually see. The free space then belongs to the editor instead of looking like a gap between a small input and the keyboard.

You can get the same view from a running debug app:

```text
ext.flutter.debugDumpRenderTree   # sizes, constraints, creators
ext.flutter.debugDumpApp          # widget types and state
```

They are developer tools, not release instrumentation, but for a layout bug they answer "which widget is actually where" faster than any screenshot.

## What the two fixes have in common

Both bugs came from treating a visual change as a small change.

- The expand toggle looked like a style swap, but it replaced a stateful element. Flutter cares about identity.
- The keyboard animation looked like a platform detail, but it is a per-frame constraint change. Flutter rebuilds whatever subscribed to it.

The invariants are portable:

1. **Keep the element chain stable across a mode switch.** Change siblings and properties, not the widget type above the stateful leaf.
2. **Do not subscribe the screen to an animation.** Measure the part that needs the geometry, cache booleans that only flip at thresholds, and let layout do the rest.

## FAQ

### Why does the keyboard close when a Flutter input expands?

The expand path probably replaces the widget above the `TextField`. When the old `EditableText` is disposed, it closes its input connection, which hides the software keyboard. Keep the composer content at the same tree position and type; change only its frame and background.

### Why does a Flutter keyboard animation cause jank?

An inset animation changes the body constraints on every frame. A `LayoutBuilder` or a `MediaQuery` dependency high in the tree subscribes the whole screen to those changes. Move the measurement down to the widget that needs it, and cache threshold decisions instead of re-reading the inset in `build`.

### Does `TextField(expands: true)` make the text fill the height?

It makes the field fill its parent. A multiline `EditableText` still lays out at its content height, so the field's surface is what expands. That is why the visible surface and the alignment of the draft matter as much as the raw height.

The composer now expands without a flash, keeps the keyboard, and fills the space in a way you can see. The keyboard animation no longer has a reason to rebuild the chat, so the next tap on the input is just a tap — and not a test of the phone's patience.
