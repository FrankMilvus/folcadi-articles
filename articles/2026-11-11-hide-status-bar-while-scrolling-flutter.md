---
layout: post
title: "Hide the status bar while scrolling in Flutter without layout jumps"
description: "Direction-aware auto-hide for a chat's app bars and the Android status bar, plus the edge-to-edge traps behind Flutter's SystemUiMode."
tags: showdev, flutter, android, ux
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-11-hide-status-bar-while-scrolling-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-11-hide-status-bar-while-scrolling-flutter.png
---

A chat screen is mostly a reading surface. On a phone, the system status bar, the floating conversation header and the pinned-message bar can eat well over a hundred density-independent pixels at the top of the screen. That is fine while you are writing a message. It is exactly the wrong use of space while you are scrolling back through a year of conversation looking for one link.

So the bars in our Flutter chat now behave like a browser's address bar: they leave when you scroll away from the newest message and come back when you scroll toward it. Two independent switches control what leaves — the in-app app bars (on by default) and the Android/iOS status bar (off by default, because the clock and battery are information most people want to keep).

The interesting part is not the fade. It is the three traps underneath it: a list that must not move when the chrome disappears, an Android status-bar API that is the wrong tool for an app targeting Android 16, and a layout inset that collapses the moment the system bar hides.

## The rule: leave on the way back, return on the way forward

A timer is the obvious first attempt: hide on scroll start, show again 900 ms after the scroll ends. It works, and it is what we shipped first. It also has a flaw you notice while reading: the bars pop back over the message you are in the middle of.

Direction is the better signal. In a chat, the newest message is the bottom of the list and the oldest is the top. We hide the bars once the user has moved about 12 px towards older history, and bring them back once they deliberately move about 24 px back towards the newest message. The asymmetric thresholds matter: the return trip needs a little more intent so hand jitter during a slow read cannot flicker the chrome.

Two more cases should reveal the bars immediately:

- the list reaches the newest message (offset zero in a reversed list);
- the composer takes focus, because typing is a deliberate interaction even if no scroll happened.

Only a finger-driven scroll counts. Programmatic scrolls — a new message, jump-to-latest, opening a search hit — must never blank the interface. Flutter tells those apart cheaply: a `ScrollStartNotification` from a drag carries `dragDetails`, a driven scroll does not. A small state machine then turns the deltas into one decision at a time:

```dart
enum BarsAction { hide, reveal }

class DirectionGate {
  DirectionGate({this.hideAfter = 12, this.revealAfter = 24});

  final double hideAfter;
  final double revealAfter;

  double _towardsOlder = 0;
  double _towardsNewest = 0;

  void newGesture() {
    _towardsOlder = 0;
    _towardsNewest = 0;
  }

  BarsAction? onDelta(double delta, {required bool hidden}) {
    if (delta == 0) return null;

    // Reversed list: positive delta moves towards older history.
    if (delta > 0) {
      _towardsNewest = 0;
      _towardsOlder += delta;
      if (hidden || _towardsOlder < hideAfter) return null;
      return BarsAction.hide;
    }

    _towardsOlder = 0;
    _towardsNewest += -delta;
    if (!hidden || _towardsNewest < revealAfter) return null;
    return BarsAction.reveal;
  }
}
```

Wiring it up is one notification listener around the message list:

```dart
NotificationListener<ScrollNotification>(
  onNotification: (notification) {
    // A horizontal gallery inside the list reports a deeper depth.
    if (notification.depth != 0) return false;

    if (notification is ScrollStartNotification) {
      _userScroll = notification.dragDetails != null;
      _gate.newGesture();
    } else if (notification is ScrollEndNotification) {
      _userScroll = false;
    } else if (notification is ScrollUpdateNotification && _userScroll) {
      final delta = notification.scrollDelta;
      if (delta == null) return false;
      switch (_gate.onDelta(delta, hidden: _barsHidden)) {
        case BarsAction.hide:
          _hideBars();
        case BarsAction.reveal:
          _showBars();
        case null:
          break;
      }
    }
    return false;
  },
  child: messageList,
)
```

If your list is not reversed, the sign flips; what matters is that positive means "towards the content you are digging for".

## Fading an overlay costs no layout

The floating header in our chat is not a real app bar. It is an overlay inside a `Stack` above a message list that already runs the full height of the body. That fact is the whole reason the animation is cheap: fading the header away reveals the messages that were behind it all along. Nothing needs to resize, no scroll offset changes, and there is no reflow at the moment the chrome disappears.

The fade is an `AnimatedOpacity` plus a fixed-pixel slide, with `IgnorePointer` once it is hidden so a tap cannot land on an invisible button:

```dart
IgnorePointer(
  ignoring: hidden,
  child: AnimatedOpacity(
    opacity: hidden ? 0 : 1,
    duration: const Duration(milliseconds: 220),
    curve: Curves.easeOutCubic,
    child: TweenAnimationBuilder<double>(
      tween: Tween(begin: 0, end: hidden ? -statusBarHeight : 0),
      duration: const Duration(milliseconds: 220),
      curve: Curves.easeOutCubic,
      builder: (_, dy, child) => Transform.translate(
        offset: Offset(0, dy),
        child: child,
      ),
      child: header,
    ),
  ),
)
```

Two details matter more than they look.

**Slide by a fixed distance, not a percentage.** An `AnimatedSlide` of `-0.12` sounds harmless until the pinned bar appears and the animated column gets taller, so the header suddenly travels further. The fixed distance also matches the system bar's own travel, which is the next point.

**Not everything should slide.** The header moves up by the status bar's height. The pinned-message bar and the stacked banners fade in place. Sliding them too drags them over the header while both are semi-transparent, which reads as two cards colliding rather than one top edge retracting.

## Android 16 makes Flutter's SystemUiMode the wrong tool

The status bar is drawn by the operating system, not by Flutter, so the only question we control is *when* to ask for it and *how* our own chrome moves alongside it. The obvious API is:

```dart
SystemChrome.setEnabledSystemUIMode(
  SystemUiMode.manual,
  overlays: const [SystemUiOverlay.bottom],
);
```

On Android that call takes the legacy path. In the engine it first calls `WindowCompat.setDecorFitsSystemWindows(window, true)` — in other words, it takes the app *out* of edge-to-edge mode — and then applies the deprecated `SYSTEM_UI_FLAG_*` flags. For an app whose whole layout is built on drawing behind the system bars, that is a regression hiding inside a one-line helper. Flutter's own documentation adds a second warning: apps targeting Android SDK 16 (API 36) have this mode ignored by the platform.

So Android now goes through our existing window channel instead, with the modern insets controller:

```kotlin
"setStatusBarVisible" -> {
    val visible = call.argument<Boolean>("visible") ?: true
    runOnUiThread {
        val controller = WindowInsetsControllerCompat(window, window.decorView)
        controller.systemBarsBehavior =
            WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
        if (visible) {
            controller.show(WindowInsetsCompat.Type.statusBars())
        } else {
            controller.hide(WindowInsetsCompat.Type.statusBars())
        }
    }
    result.success(null)
}
```

This hides only the status bar, keeps the navigation bar, keeps edge-to-edge enabled, and still lets a swipe from the top reveal the bar temporarily. On iOS the Flutter overlay API is fine — it maps to `prefersStatusBarHidden` and has no layout side effects — so the Dart service picks the right path per platform:

```dart
if (defaultTargetPlatform == TargetPlatform.android) {
  await _channel.invokeMethod<void>('setStatusBarVisible', {
    'visible': visible,
  });
} else if (defaultTargetPlatform == TargetPlatform.iOS) {
  await SystemChrome.setEnabledSystemUIMode(
    SystemUiMode.manual,
    overlays:
        visible ? SystemUiOverlay.values : const [SystemUiOverlay.bottom],
  );
}
```

Wrap the channel call in a `try/catch` and log, never throw. A chat that cannot hide a clock is still a working chat.

## The status bar takes its padding with it

Hiding the status bar does not just remove pixels from the screen; it removes the inset from `MediaQuery.padding.top`. Every widget that uses the safe area — including the floating header's position and the list's top padding — suddenly moves up by 24 to 48 dp, and then jumps back down when the bar returns. If you have ever watched a header “bounce” in an immersive app, this is usually why.

The fix is to pin the last known inset for as long as the bar is hidden, and release it only after the platform reports the padding again:

```dart
final media = MediaQuery.of(context);
final pinnedTop = max(media.padding.top, _frozenTopInset);

return MediaQuery(
  data: _systemBarHidden
      ? media.copyWith(
          padding: media.padding.copyWith(top: pinnedTop),
          viewPadding: media.viewPadding.copyWith(top: pinnedTop),
        )
      : media,
  child: child,
);
```

Release it late. Clearing the flag on the same frame as the `show` call leaves one frame at zero top padding, which is exactly enough to see a twitch. Waiting for `MediaQuery` to report the restored padding costs nothing and removes the artefact.

Note that this pinning is only needed for the *system* bar. The app-bars switch never touches the inset because the chrome is an overlay; that is why the two switches can be independent.

## Making two animations look like one

The system bar's animation is owned by Android or iOS. You cannot drive its frames from Flutter, and its duration varies by platform and OEM. What you can do is match the three things a viewer notices:

1. **Start on the same beat.** Send the platform hide/show call *before* the Flutter `setState`, not after. The channel message is then queued first and the platform animation does not visibly trail your fade.
2. **Match the duration and easing.** The chrome uses 220 ms with `Curves.easeOutCubic` in both directions, close to the platform's inset animation. If a device feels different, this is the one number to tune.
3. **Match the distance.** The header slides by exactly the status bar's height, the distance the system bar itself travels, so the gap between them stays constant during the motion.

The second switch matters here: with the system bar off (the default), only the app bars move, and there is no platform animation to match.

## The edges: when not to hide

Auto-hide must lose gracefully to modes that own the top edge. In our chat, the bars stay put while search or selection mode is active, while the header menu is expanded, and while the composer is in its large editor mode. Opening search or selection also restores the bars immediately.

And because the hidden state is UI state, it needs deterministic exits. The bars are restored on:

- leaving the chat (`dispose`);
- pushing another route on top (a profile, the media viewer, a call);
- the app leaving the `resumed` lifecycle state — background, app switcher, system dialog;
- turning either switch off while its bars are hidden;
- the composer gaining focus or the list reaching the bottom.

Everything is idempotent, so two restore triggers firing together are harmless.

## What the tests cover

Three layers, each cheap to run:

- **The direction gate** is pure logic, so it gets ordinary unit tests: thresholds, accumulation across several updates, a direction flip resetting the other counter, a new gesture forgetting a partial move, and no repeated actions while already in that state.
- **The platform contract** is mocked at the method-channel boundary: hiding and showing must send `setStatusBarVisible` with the expected `visible` argument, so a rename on either side fails a test instead of silently doing nothing on a device.
- **The settings page** is pumped on a simulated Android target to prove both switches are reachable, and on a desktop target to prove they are absent (the feature is mobile-only; desktop windows have the room and no system bar).

None of these replace a real device, because the thing that broke first — the system bar API — is exactly the part a widget test cannot see.

## The short version

- If the chrome is an overlay, animate it; do not change layout to make room for the animation.
- Use direction, not a timer. Small threshold to leave, larger one to return.
- On Android, hide the status bar with `WindowInsetsController`, not with the legacy `SystemUiMode` overlays, especially with a modern target SDK.
- Pin `MediaQuery.padding.top` while the bar is away and release it only when the new padding arrives.
- Start the platform call before your own animation, and match the duration and distance so the two read as one movement.
- Let search, selection, menus and the composer win. Auto-hide is a convenience, not a mode.

## FAQ

**Why not `immersiveSticky`?** It also hides the navigation bar, and on Android it takes the same legacy overlay path. We wanted the navigation controls to stay, and the modern insets controller is a better fit for an app that is already edge-to-edge.

**Does this work on iOS?** Yes. Hiding the status bar through Flutter's overlay API maps to `prefersStatusBarHidden`, and the home indicator is left alone.

**Does it work on desktop?** Not by design. The switches are shown only on Android and iOS; a desktop window has plenty of room, and there is no system status bar in the same sense.

**Does the hidden status bar break safe areas?** No. The chat pins the last known top inset while the bar is gone, so the notch and the header keep their relationship instead of collapsing to zero.

**What about accessibility?** The chrome is wrapped in an `IgnorePointer` and fades to zero opacity, so invisible buttons cannot be tapped and the hidden chrome leaves the semantics tree. The direction thresholds also prevent flicker, which matters more than most animation tweaks for motion sensitivity.
