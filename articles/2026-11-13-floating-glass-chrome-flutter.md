---
layout: post
title: "One floating glass header for every screen in a Flutter app"
description: "A floating glass app bar Flutter screens can share: one measured clearance, a 30 % tint, and the sync bug that turned the frost back into a solid chip."
tags: showdev, flutter, ux, design
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-13-floating-glass-chrome-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-13-floating-glass-chrome-flutter.png
---

A chat header, a saved-notes page and both profile screens now render the same floating glass app bar. Not a similar one: the same card, the same 12 dp side margins, the same 8 dp gap under the status bar, the same stadium radius the navigation dock already used. Opening a profile from a conversation used to feel like arriving in a different app, because a stock Material bar sat flat across the top of the page while everything under it floated.

Three surfaces forced a better question than how to style an app bar: what does a floating bar need in order to be correct? It came down to a tint value, a measured spacer and one sync bug that only appeared on the next launch.

## The bar is a card, not a strip

The conversation header is 64 dp tall and holds a 36 dp circular back control, a hairline pill containing the avatar and name, and the same 36 dp circles for its actions. All of it sits on one `ChromeGlassSurface`, inside `Positioned` bounds that keep 12 dp of air on each side and 8 dp below the status area.

The corner radius is not a fresh constant. `glassBarRadius` reads the app's configurable card radius and clamps it to half the bar's height, so a user who likes very round cards cannot turn the header into an oval, and the bar cannot drift away from the dock when that setting changes.

My Space is the exception that proves the rule: it is 48 dp tall and its controls are the app's mini floating buttons rather than full circles, because that page has no avatar, no presence line and no call action to display. Same glass, same margins, same gap, same radius function — just a shorter row and less inside it.

Because the card floats, its blank pixels are not decoration. They are hit-test area. A `Listener` with `HitTestBehavior.opaque` wraps the surface, so a tap in the gap between the name and the call button does nothing instead of opening the photo underneath.

## Clearance: the number that makes it look intentional

A floating bar over a scroll view has exactly two failure modes. Either the content starts too high and hides under the frost, or it starts too low and leaves a band of empty space that reads as a mistake. There is no third option, and no amount of visual polish hides the wrong one.

The fix is to measure the bar once and reserve exactly that much:

```dart
double glassAppBarClearance(BuildContext context) =>
    MediaQuery.paddingOf(context).top +
    kChromeGap +
    kGlassAppBarHeight +
    kChromeGap;
```

Status bar, gap, bar, gap. That is the leading spacer sliver of the page's scroll view. Content begins below it, and then scrolls up behind the bar as the user moves, which is the effect that makes the glass worth having.

Use the value as a leading sliver rather than as padding on the first card. Padding travels with its child: pull to refresh or insert a banner and the spacing goes with it. A spacer sliver is geometry, not content.

The horizontal gutter follows the same logic. A fixed 24 dp gutter next to a bar with 12 dp margins reads as unused space on a phone, so `glassPageGutter` returns 12 dp below 600 dp of width and keeps 24 dp on wider windows where the extra room helps line length. A bar that is inset while the text under it is not looks like two unrelated layouts.

## Content has to run underneath, or there is nothing to frost

The bar is a sibling in a `Stack`, not a widget above the scroll view in a `Column`:

```dart
Stack(
  children: [
    Positioned.fill(child: scrollView),
    Positioned(
      top: 0,
      left: 0,
      right: 0,
      child: IgnorePointer(
        ignoring: hidden,
        child: AnimatedSlide(
          offset: hidden ? const Offset(0, -1) : Offset.zero,
          duration: duration,
          child: AnimatedOpacity(opacity: hidden ? 0 : 1, child: appBar),
        ),
      ),
    ),
  ],
)
```

This is why the profile pages use a `CustomScrollView` that fills the body rather than a `SliverAppBar`. A sliver app bar is part of the scroll view's layout: the viewport ends where the bar ends, so there is nothing behind the bar to blur. Once the bar moved into a stack, the map at the top of a profile scrolled under it and the frost finally had content to work with.

Retraction therefore costs nothing: hiding an overlay resizes nothing and never touches the scroll offset, so there is no reflow at the moment the chrome leaves.

## The floating glass app bar trap: never tint it with its own background

Here is the mistake worth the whole article. The chat background is the page surface plus a wallpaper pattern at about 25 % opacity. The obvious tint for a glass bar on that page is the page background colour at 50 % or 60 %. It looks correct in isolation, and on the device the bar reads as a solid plastic chip.

The reason is arithmetic. A `surface` fill at 50 % over a `surface`-coloured page does not tint anything: it flattens the wallpaper pattern from roughly 25 % to roughly 11 % and then the blur has almost no contrast left to work with. The `BackdropFilter` is still running. You just cannot see it.

The header and the pinned-message preview shipped that way while the composer shipped with a lighter tint, and the two looked like different materials sitting 300 dp apart. The fix was to standardise on the shared chrome tint:

```dart
Color glassTint(BuildContext context) {
  final scheme = Theme.of(context).colorScheme;
  return scheme.surface.withValues(
    alpha: Theme.of(context).brightness == Brightness.light ? 0.30 : 0.26,
  );
}
```

Around 30 % in light mode, 26 % in dark, painted above the blur so text stays crisp. Tint higher only when text genuinely needs more contrast — the chat action menu sits at 60 % because it is a dense grid of labels, and it is still readable as glass.

Two related rules fell out of the same debugging session. A frame drawn on top of the glass should draw no fill and no blur of its own: the avatar-and-name pill uses `ChromeOutline`, a hairline with nothing inside it, because a second tinted patch made the chip denser than the bar around it. And the border must be painted, not laid out. A `foregroundDecoration` (or a `DecoratedBox`) hairline adds no size; a layout border grows the card by its width and silently desynchronises any reserved slot.

## Retraction without a sliver

The profile pages wanted the old `SliverAppBar(floating: true, snap: true)` feel: the bar leaves when you scroll down and returns when you scroll up. The replacement is a small host widget wrapped around the page body. It listens to the scroll notifications, ignores anything with a non-zero depth so a horizontal gallery inside the page cannot blank the bar, and feeds the deltas to the same direction tracker the chat uses.

The tracker is asymmetric on purpose. About 12 dp of downward travel hides the bar; about 24 dp of upward travel brings it back. Reaching the very top always restores it. The extra intent required to return is what stops a slow, jittery read from flickering the chrome.

Mouse wheel and trackpad scrolls count, not just touch drags. That matters on Windows and Linux, where a touch-only detector would have shipped a feature that never fired for most desktop users.

The retraction here is independent of the chat's "Hide app bars while scrolling" setting. The chat setting is about message bars and is its own toggle; a profile page simply behaves like a browser page. Sharing the tracker, not the setting, keeps those two intents separate.

## The status bar lost its publisher

`AppBar` quietly publishes the status-bar icon brightness for the page it is on. Remove the `AppBar` and the system falls back to whatever was last set, which in a light theme usually means white icons on a white page — an invisible clock.

The replacement is one `AnnotatedRegion` around the page:

```dart
AnnotatedRegion<SystemUiOverlayStyle>(
  value: Theme.of(context).brightness == Brightness.light
      ? SystemUiOverlayStyle.dark
      : SystemUiOverlayStyle.light,
  child: Scaffold(body: GlassAppBarHost(appBar: bar, child: scrollView)),
)
```

My Space needed the same fix when it dropped its docked bar. It is not polish: it is the difference between a status bar and a rumour, and no widget test can see it, because the harness has no system status bar.

## The refresh spinner was frostbitten

`RefreshIndicator` places its spinner a fixed distance from the top of its scroll view. With a floating bar, that distance is inside the bar, so the spinner appeared behind the frost — a pale ring smeared under the glass, which looked like a rendering bug rather than a refresh.

The fix is one value, `glassAppBarClearance(context) + 20`, so the indicator settles below the bar instead of underneath it. The extra 20 dp is the spinner's own breathing room; the clearance handles the chrome.

## The bug that flipped the glass back to solid

Everything above was visual. The nastiest issue was not.

The appearance page batches settings changes and pushes them to the account after a short debounce. When the page was disposed, the pending timer was cancelled. Toggle a setting, leave the page within that window, and the change stayed local while the account kept the old value. On the next launch the local preference painted the first frame, then the profile load applied the stale remote value, and the chrome became solid again. From the user's side the app simply forgot the choice overnight.

Three fixes:

- Disposing the page now flushes a pending push before cancelling the timer.
- The two transparency switches push immediately, so leaving the page cannot drop them.
- The choice is marked dirty locally until a push succeeds. While dirty, startup keeps the local value, then pushes it. A one-time migration marks an existing local choice dirty, because the marker did not exist before.

Then the second writer surfaced. A background sync of the quiet-hours offset merged that offset into a settings map it had fetched earlier and pushed the whole blob back. If the fetched map predated a local chrome change, the push overwrote it or dropped it. Now the local settings are merged over the fetched map, so local values win and unknown keys survive. The dirty marker key is versioned as well, so a marker cleared by the old path is ignored instead of being trusted.

The transferable rule: a preference that paints instantly and syncs later needs a dirty flag and a write path that never rebuilds the document from a stale copy. Debouncing the push is fine. Debouncing it without a flush on dispose is how a setting becomes a rumour that it works.

## What the tests can and cannot see

The pure parts are cheap to test: the direction tracker's thresholds and resets, the host's retract and restore, the phone and wide-window gutters, the tint and its transparent branch. The parts that broke in the field — status-bar brightness, frost over a real wallpaper — are invisible to a widget test, which is why the tint bug shipped: the code did exactly what it was told.


## Rules I would keep

- One chrome component, not three styled app bars. Divergence is what makes a profile page feel like a different app.
- Reserve the bar's height with a measured spacer; match the gutter to its margins.
- Put the scroll view in a `Stack` under the bar, so there is content to frost.
- Never tint glass with the surface it sits on. Around 30 % light, 26 % dark.
- Hairlines belong in a foreground decoration, never in layout.
- If a setting paints now and syncs later, give it a dirty flag and flush on dispose.

The bar itself is now a reusable `GlassAppBar` plus a host, which means the next page that needs chrome does not need a design decision — only a scroll view and a clearance. If you are migrating pages one at a time, the [auto-hide chrome work](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-11-hide-status-bar-while-scrolling-flutter.html) covers the scroll-direction details, and the [on-device translation pass](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-04-before-babel-on-device-translation.html) shows the same rule from the data side: the visible behaviour is the contract, the internals are allowed to change.
