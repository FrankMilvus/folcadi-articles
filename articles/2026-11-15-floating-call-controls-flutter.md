---
layout: post
title: "Call controls that match the app: one floating-control component"
description: "A Flutter floating call controls pattern: one circular button across three call surfaces, plus the keyboard inset that keeps the mini window visible."
tags: showdev, flutter, ux, webrtc
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-15-floating-call-controls-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-15-floating-call-controls-flutter.png
---

Floating call controls in Flutter get built twice: once for the call screen, and once more for whatever surface shows the call after the user leaves that screen. Ours had been built three times. The live control row was a set of solid grey discs with a red one at the end, the end card used stock Material filled and outlined buttons, and the top-left collapse button was a semi-transparent black circle with a white arrow. Each was individually reasonable. Together they made an active call look like a different app from the one that had started it.

The fix was one circular control component with two accents, plus a keyboard problem in the collapsed window that had nothing to do with styling at all.

## Three surfaces, three dialects

The call screen is the most visually exposed state in the app: it covers everything, it usually sits over live video, and the user is holding the phone one-handed while looking at somebody's face. It is also the state where the buttons must be found without reading.

The old row was hand-rolled: a circle, a fixed colour, an icon, a gesture detector. That pattern is fine once; copied three times it produces three slightly different measurements and three ideas of what "off" looks like.

Meanwhile every other floating control — the navigation dock's destinations, the friends list actions, the profile edit button — had settled on a different vocabulary: a surface-filled circle with a two-pixel accent outline. The call screen was written before that vocabulary existed and nobody had a reason to revisit it.

## Floating call controls in Flutter should not fork

The shared piece is `CircleSurface`: a circle filled with the theme's surface colour, outlined at two pixels, with the icon tinted to match. `AppCircleButton` wraps it in an `IconButton` and splits the visible circle from the tap target around it.

```dart
Container(
  width: size,
  height: size,
  decoration: BoxDecoration(
    shape: BoxShape.circle,
    color: colors.surface,
    border: Border.all(
      color: selected ? color : color.withValues(alpha: 0.6),
      width: 2,
    ),
  ),
  child: IconTheme(
    data: IconThemeData(color: color),
    child: child,
  ),
)
```

Two properties do the real work. The fill is the theme's surface, so the circle sits on glass, video or a photo rather than vanishing into a translucent overlay. And the outline is drawn at full strength for active controls instead of the muted sixty per cent used by inactive dock items. Over a bright video call, a muted outline is an invisible button.

The call screen composes `CallActionButton`: that circle at a call-specific size, plus a semantics label. The end card uses the same widget. The collapse control uses plain `AppCircleButton` at its default size, because it belongs to the top bar rather than the thumb row.

## Why the call row is 72 dp and the dock is 40

`CallActionButton` defines one number: a 72 dp diameter, circle and tap target together. The app's mini floating button is 40 dp, and it is the right size for a list row you tap while looking at it.

A call is different. The user is often not looking at the controls at all; they are looking at the person, or at something in the room, and reaching by muscle memory. A 40 dp target over live video is a miss-tap magnet, and a miss-tap on a call row can mean hanging up on someone. Seventy-two is roughly the difference between aiming and hoping.

The glyph follows the size: 32 px icons inside the 72 dp circle, which keeps the visual weight close to the text-free dock buttons instead of turning them into tiny icons in big rings.

An audio call gets three controls, not four: mute, speaker, end. The camera switch only exists for video, so hiding it leaves the row evenly spaced rather than gapped where a fourth button would have been.

## One shape, two accents

The destructive action should be recognisable, and the three usual ways are all worse than colour. A different shape breaks the row's rhythm; a bigger button makes the row unstable; a different material makes it look borrowed from another component library.

So the end call keeps the exact circle, outline weight and icon size, and changes only the accent: the theme's error colour replaces the primary one. A muted microphone does the same, because "your mic is off" is also a state the user needs to notice at a glance. Everything else stays primary. Shape carries identity; colour carries state.

This is why the accent is an optional parameter on the shared surface rather than a second widget. The dock, the chat header and the send button use the primary default, so the override touched nothing they rely on.

Accessibility labels did not change, deliberately. `Mute` and `Unmute`, `Switch camera`, `Speaker on` and `Speaker off`, `End call` — the icons are ambiguous by nature, and the labels are what makes the row usable with a screen reader. A visual refactor that quietly renamed them would be a regression dressed as a redesign.

## The end card moves to where the thumb already is

The end card placed its two actions mid-column: a filled button and an outlined one under the call summary. Legible, and in the wrong place — the hand that just ended a call is at the bottom of the screen.

Both actions are now `CallActionButton`s in the same bottom row, with the same 24 dp padding and the same `spaceEvenly`, so redialling or leaving occupies the coordinates the user's thumb already knows. The icons are a phone and an arrow; the words "Call again" and "Back" survive as semantics labels rather than as visible text.

One ordering detail matters more than it sounds. The row is painted above the card's dismiss layer, so a press on either button acts immediately. If it sits below that layer, the first tap only dismisses the overlay and the user has to tap twice — a bug that looks like latency.

## The collapsed window keeps its own smaller pair

The floating window an active call collapses into is a deliberately different surface. It is 132 by 180 logical pixels, it floats over whatever the user is doing, and it holds a peer video, a name, a running duration and two quick actions: mute and hang up.

A 72 dp circle does not fit in that card, and pretending otherwise would eat the video. So the window keeps two compact 36 dp discs. They use the same red for hang-up and the same semantics labels, and the mute button turns red when muted. It is the one place where the call controls are not the shared component, and that is a size decision rather than a style one.

The more interesting part of that window is its placement.

## The keyboard is occupied space

Collapse a call, go back into a chat, and tap the composer. The software keyboard rises — and the floating call window was still where it had been left, sitting under the keyboard with its buttons and the peer's video hidden behind the keys.

The window lives above the navigator, mounted at the app level so it survives route changes while a call is running. That also means the keyboard never resizes it: the inset belongs to a different subtree, and the window has to notice and move itself.

The fix starts by treating the keyboard as part of the area that is no longer available:

```dart
final padding = MediaQuery.paddingOf(context);
final keyboard = MediaQuery.viewInsetsOf(context).bottom;
final bottomInset = keyboard > padding.bottom ? keyboard : padding.bottom;
```

That `bottomInset` is subtracted from the height the window may occupy, so the top-left corner it is allowed to reach stops where the keyboard begins. Dragging, clamping and edge snapping all keep working against the reduced rectangle.

The comparison is deliberately `max`, not a sum. The keyboard inset already extends down through the system's bottom padding, so adding the two would float the window far higher than the keyboard's actual top edge — technically clear, visually wrong, and worst on phones with a large home-indicator area.

Because both insets are read as `MediaQuery` dependencies, the window rebuilds as the keyboard animates and the safe area changes — no listener to leak, no controller to sync with an animation the system owns.

## A lift that is not a position

The subtle half of the fix is that the lift must not be remembered.

The window stores its position as a fraction of the usable area — which side, how far down — and restores it on the next call. If the keyboard-driven lift wrote into that stored value, a user who typed one message mid-call would come back to a window pinned to the middle of the screen forever. The remembered position would be a fossil of a transient layout.

So the lift is display-only. The stored position is written by exactly one thing: a drag. While the keyboard is up, the window is resolved from its remembered fraction inside the shrunken area, which naturally puts it above the keyboard. When the keyboard closes, the area grows back and the window returns to the side and fraction it remembered.

Dragging while the keyboard is visible is the deliberate exception. That is a real decision by the user about where the window should live, so it saves the new spot — and it is saved as a fraction of the temporarily smaller area, which means it lands somewhere sensible in both states.

Because the remembered position is a fraction of the usable area rather than a pixel coordinate, the window lands proportionally: near the bottom of that area it lifts clear of the keyboard, and near the top it barely moves. Edge snapping still happens on release, measured against the reduced width.

## The test that fakes a 300 px keyboard

Widget tests cannot summon a real keyboard, but they can set the inset that one produces. The test sets the device pixel ratio to one so physical and logical pixels agree, sizes the surface, pumps the overlay the same way the app mounts it, and records the window's rectangle.

It then asserts the bottom edge at one margin above the safe area in the default bottom-right spot, sets a 300 px bottom inset, and asserts that the window's bottom edge has moved above the keyboard — still one margin clear. Clearing the inset returns it to the original rectangle, which is the part that proves the lift was display-only rather than a new remembered position.

That last assertion is the one I would keep in any app with a floating surface. "It moves correctly" is easy to get right by accident; "it goes back" is the part that regresses silently.

## The short version

- One circle, one outline weight, one icon size: identity comes from shape, state comes from accent.
- Size the row for the hand, not for the grid. 72 dp over live video is a feature; 40 dp is a gamble.
- Keep semantics labels stable through a visual refactor; they are part of the contract.
- Put the post-call actions where the controls were, and paint them above any dismiss layer.
- Treat `viewInsets.bottom` as occupied space, and inset by `max(bottom padding, keyboard)` rather than their sum.
- Never let a transient layout push write into remembered state; drag is the only author of position.
- Test the return trip, not just the lift.

Both halves of this came from the same question: what does this surface need in order to be correct? For the buttons it was a size and a shape. For the window it was a rectangle that shrinks and a position that does not. The floating dock taught us the same thing about chrome: geometry first, then colour.

If you have a floating surface that has to survive a keyboard, I would like to hear which half bit you — the inset arithmetic or the remembered position.
