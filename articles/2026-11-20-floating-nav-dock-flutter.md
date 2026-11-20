---
layout: post
title: "A floating glass dock: a Flutter nav bar that never spans the screen"
description: "A floating navigation bar in Flutter, rebuilt as a 420 dp glass dock: one clearance token, foreground hairlines, FAB lifts and a clear mode."
tags: showdev, flutter, design, ux
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-20-floating-nav-dock-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-20-floating-nav-dock-flutter.png
---

A floating navigation bar in Flutter is usually a rounded `BottomNavigationBar` with a shadow and a bit of padding. Ours is a 420 dp glass pill, lifted off the bottom edge, with a truly transparent background so a `BackdropFilter` can frost whatever scrolls underneath it. On a phone it is a compact centred card; on a 1920 dp desktop window it stays exactly the same width instead of stretching into a stock-looking strip.

The interesting part is not the pill. It is everything the pill drags with it: a safe-area inset that was being counted twice, a border that changed the card's height by one pixel, floating buttons in nested pages that fell behind it, and a scroll reserve that had to collapse when the keyboard appeared.

## The bar became a dock

The old bar was full width, docked to the edge, with a border on top and square bottom corners. It was the one piece of chrome in the app that ignored the floating-card language used by the chat bars. The replacement keeps the navigation behaviour identical and changes only geometry, which is a good trade: users notice the shape instantly and never notice that nothing else moved.

Four tokens define it:

- `kNavDockHeight` is 64 dp: a 56 dp icon row plus 4 dp of padding above and below.
- `kNavDockMaxWidth` is 420 dp. The dock is `min(windowWidth - 24, 420)` wide and centred.
- `kNavDockSideMargin` is 12 dp, the same margin the chat header uses.
- `kNavDockGap` is 8 dp above the device's bottom inset.

The inset itself is never painted inside the card. The dock is lifted by `inset + gap` as a bottom padding, so the rounded shape always sits on the content rather than being clipped by the home-indicator strip.

The corner radius reuses the app's configurable card radius, clamped to half the dock's height. A user who sets very round cards gets a stadium; the clamp stops an extreme value from turning the dock into an oval.

## No fill, no shadow, no elevation

`GlassSurface` defaults to no fill and no drop shadow. For the dock, that is not a stylistic preference — it is the feature. A fill would hide the content the blur is meant to soften, and a box shadow behind a transparent interior shows through as a grey haze around the buttons.

So the Material bar underneath is stripped of its own appearance:

```dart
NavigationBar(
  height: kNavDockIconRowHeight,
  backgroundColor: Colors.transparent,
  indicatorColor: Colors.transparent,
  shadowColor: Colors.transparent,
  elevation: 0,
  surfaceTintColor: Colors.transparent,
)
```

Everything visible comes from the glass surface, the hairline, the destination buttons and the active orb. That is also why the built-in indicator is transparent: Material's indicator can only paint a flat fill, and the dock wants a soft, glowing orb that can slide between destinations.

The cost is real and worth stating: a backdrop filter over a 420 × 64 pill forces the backdrop to be saved every frame while the page scrolls. If a low-end Android device shows jank, lower the blur sigma before you consider adding a fill. Adding a fill fixes the jank by removing the effect you wanted.

## The safe-area inset was being counted twice

`NavigationBar` wraps itself in a `SafeArea`, which adds the bottom inset inside the pill. The dock also adds that inset, as its bottom margin, to keep the card off the home indicator. Left alone, the card grew by the inset a second time and the reserved slot no longer matched the card.

The fix is to remove the copied padding before the bar sees it:

```dart
MediaQuery.removePadding(
  context: context,
  removeLeft: true,
  removeRight: true,
  removeBottom: true,
  child: NavigationBar(...),
)
```

Left and right go too, for a different reason. The pill is already inset and centred inside the window, so a display cutout's side inset would squeeze the destinations and desynchronise the orb's measured position. The dock owns its horizontal geometry; the system inset should not get a second vote.

## One clearance for a floating navigation bar in Flutter

There are two places a page can live: inside the four-tab shell, or inside the shell used for pushed pages such as Settings and About. Both now use the same `Scaffold`, with `extendBody: true`, the same `NavDockBody` wrapper and the same dock. That is what makes the dock identical on every screen instead of only on the tabs.

`NavDockBody` raises the body's `MediaQuery` bottom padding to the clearance and inserts a `NavDockScope`:

```dart
double navDockClearance(BuildContext context) {
  final media = MediaQuery.of(context);
  if (media.viewInsets.bottom > 0) return 0;
  return media.viewPadding.bottom + kNavDockGap + kNavDockHeight;
}
```

It reads `viewPadding`, not `padding`, because `padding` is the value the shell is about to overwrite. Reading the physical inset means the function returns the same number on both sides of the override, which matters for tests and for any page that computes its own spacing.

The scope is the other half. A page pushed without a shell — a settings page opened directly from a chat, for example — has no dock, so reserving the dock's space would leave a phantom gap at the bottom of its list. `NavDockScope` lets those pages ask whether a dock is actually present and reserve nothing when it is not.

## The border that changed the height by a pixel

This one is small, annoying and easy to ship. A border added as layout decoration grows the box by its width. The dock's reserved slot is computed from the tokens, so a 1 px border made the card 65 dp tall while the clearance still said 64 dp. The result is a one-pixel mismatch between what the shell reserves and what the dock occupies — the kind of thing that turns into a stray scroll offset or a clipped edge.

Paint the hairline in a foreground decoration instead. It draws over the card without contributing to layout, so the measured height stays exactly the token value. The same rule applies to the chat bars: any hairline that participates in layout will eventually desynchronise a number that someone else computed.

A widget test now compares the dock's rendered height with `navDockClearance` and allows half a pixel of rounding. That test is boring, and it is the reason the bug did not come back.

## FABs sit in the dock's shadow

Once the shell extends its body behind the dock, a floating action button inside a nested page `Scaffold` lands at the bottom of the window — directly behind the dock. The Friends page has a stack of them, the Profile page has an edit button, and the user-management page has its own.

The obvious fix is to lift each one by the full clearance. That overshoots. `Scaffold` already positions a floating button above the system inset, so adding the full clearance lifts the button one inset too high and leaves the list's end spacer short by the same amount. The lift should be the dock's own contribution:

```dart
double navDockFabLift(BuildContext context) {
  if (!NavDockScope.isPresent(context)) return 0;
  final clearance = navDockClearance(context);
  if (clearance == 0) return 0;
  return clearance - MediaQuery.viewPaddingOf(context).bottom;
}
```

Two off-by-one-inset bugs in one feature is a pattern, not bad luck: whenever a widget and a framework both apply a system inset, one of them has to own it explicitly.

## MediaQuery padding only reaches lists that ask

The shell's `MediaQuery` override is not magic. A `ScrollView` with a null `padding` reads `MediaQuery.padding` and applies it, so those lists clear the dock for free. A list that sets its own padding ignores the inherited value entirely.

That distinction is easy to miss because most pages in a settings tree set explicit padding. Each one now wraps its padding in a helper:

```dart
padding: navDockPadding(context, const EdgeInsets.all(16))
```

The helper adds the clearance to the bottom only when a shell is present. Search was the visible casualty: it kept a `SafeArea` that consumed the injected bottom padding, so the scroll viewport ended above the dock and the frosted glass had nothing behind it. It now uses `SafeArea(top: false, bottom: false)` and clears the dock through the helper, so results scroll under the glass like every other tab.

## The clear mode removes things

The dock's setting is called "Transparent navigation bar". It does not add a lighter fill or a softer blur; it removes the `BackdropFilter` and the hairline border, leaving only the destination buttons and the orb. Content behind the dock becomes completely sharp, which is what the person asking for the option actually wanted.

It is also cheaper: no backdrop is saved per frame in that mode. Both states are fill-free, so the switch is between frosted and perfectly clear rather than between two tints.

The choice is cached locally, read before the first frame so the dock never flashes frosted on launch, and synced to the account so it follows the user to another device. Reset returns it to frosted. If a page ever needs a legibility guarantee while the clear mode is on, tint the parent of the blur rather than putting a fill behind it — a fill is the one change that undoes the whole effect.

## Buttons that match the rest of the app

The dock used bare Material icons while every other floating control — the Friends button stack, the Profile edit button — was a circular outlined button. It now speaks the same language: Search, Friends and Profile are 44 px circles with a surface fill and a 2 px primary ring, with a 24 px icon inside. Unselected destinations use the ring at reduced alpha; the selected one uses the full-strength ring and the active accent.

More keeps the bare brand mark, at 48 px. It is slightly larger than the circles so the logo's internal padding still reads level with them, and it deliberately has no ring, because it is the entry point to everything else rather than a peer destination.

The active orb still slides behind the selected circle, so selection reads as a soft halo around a button. After the preview, the glide moved from 300 ms to 600 ms: next to outlined circles, 300 ms read as a snap rather than a movement. The Motion setting still scales or disables it like every other animation.

The trade-off is honest: the opaque circle paints above `NavigationBar`'s ink layer, so the old tap ripple is no longer visible. Haptics, the orb flight and the small pop-in on the selected icon carry the feedback instead.

## The follow-ups that mattered

Four defects surfaced after the first version, and all of them were about ownership rather than appearance.

- **Clearance outside a shell.** Every page assumed a dock existed. `NavDockScope` made the reserve conditional, so pages pushed on plain routes stopped reserving space for something that was not there.
- **FAB lifts.** Subtracting the `Scaffold` inset from the lift fixed both the button position and the list's end spacing in one change.
- **Keyboard.** `navDockClearance` returns zero while the software keyboard is up, because the dock is hidden behind it and reserving its height would leave dead scroll area above the IME.
- **Persistence.** The transparency choice round-trips through the local cache and the account settings, including a reset path, so the dock does not change appearance between launches or devices.

## What I would steal next time

- One geometry file shared by the widget and every shell, so nobody re-derives the height.
- A clearance function that reads `viewPadding`, so it is stable on both sides of a `MediaQuery` override.
- Foreground hairlines everywhere. A border in layout is a changed size.
- An explicit scope for "is this page hosted by the shell", instead of assuming every route has a dock.
- A clear mode that removes the effect rather than imitating it with a colour.

The shell work pairs well with the [tab-shell keep-alive pattern](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-19-flutter-tab-shell-keep-pages-alive.html) if you are changing how tabs are hosted, and with the [auto-hiding chrome](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-11-hide-status-bar-while-scrolling-flutter.html) if your floating bars also need to retract. The pattern is the same in both: measure the chrome once, hand the number to the content, and let the overlay float above it.
