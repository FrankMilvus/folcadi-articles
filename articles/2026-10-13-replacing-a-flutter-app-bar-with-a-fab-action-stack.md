---
layout: post
title: Replacing a Flutter app bar with a FAB action stack
description: "How we rebuilt the GwinG notification center without a toolbar: a FAB action stack, a zero-height AppBar, unique hero tags, and greyed-out disabled actions."
tags: flutter, ux, showdev, webdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-replacing-a-flutter-app-bar-with-a-fab-action-stack.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-13-replacing-a-flutter-app-bar-with-a-fab-action-stack.png
---

The **GwinG** notification center is a short screen: reactions, profile views
and group events, most recent first. It opened from the Friends header's overflow
menu and carried a perfectly ordinary `AppBar` — a "Notifications" title on the
left, a **mark all read** and a **clear all** icon button on the right.

It worked. It also looked like every other screen, and it put the three things
you actually do on that screen — leave it, mark it read, clear it — into small
icon buttons at the very top of the phone, the hardest place to reach.

So we deleted the app bar. In this post I'll walk through what replaced it: a
vertical **floating action button** stack, why a Scaffold with no `AppBar` loses
its managed status bar (and the zero-height trick that gets it back), and the two
gotchas that bite the moment you put more than one FAB on a route.

## Why take the app bar out

Three reasons, in order of how much they actually mattered:

- **Reachability.** On a phone, top-right icon buttons are a stretch. The bottom
  third of the screen is where a thumb lives, and that is FAB territory.
- **Consistency.** The app already had a house style for floating buttons — an
  outlined circle in the brand colour — used across the Friends page, the profile
  screen and the full-screen map. The notification center was one of the few
  screens still speaking a different language.
- **Less chrome.** The title just said "Notifications" on the screen that the
  notification icon opened. It was a label telling you what you already knew, in
  the one row where vertical space is scarcest.

The plan was simple to state: drop the toolbar, and render **back**,
**mark all read** and **clear all** as the same floating buttons used everywhere
else.

## The catch: no AppBar means no managed status bar

Here is the part that surprises people. A `Scaffold` with an `AppBar` doesn't just
draw a toolbar — it also lets Flutter manage the **status bar** for you: the
system bar behind the clock and battery gets the page's colour and the right
icon brightness, consistently across every screen.

Take the `AppBar` away and that management goes with it. On Android especially,
you can end up with a status bar whose colour no longer matches the page, or
whose icons are dark-on-dark. It's the kind of thing that looks like a bug in the
OS rather than in your layout.

The fix is not to fight it — it's to keep a toolbar that draws nothing:

```dart
Scaffold(
  // A managed status bar without a toolbar. This still reserves the
  // status-bar inset and lets Flutter colour the bar and pick the icon
  // brightness, exactly as a normal AppBar would.
  appBar: AppBar(
    toolbarHeight: 0,
    elevation: 0,
    scrolledUnderElevation: 0,
    shape: const RoundedRectangleBorder(),
  ),
  body: ...,
);
```

A zero-height `AppBar` keeps everything you want from the bar — the status-bar
inset and the managed system chrome — and draws none of it. We already lean on
this same trick on the Friends page (which draws its own custom header below) and
on the full-screen map, so the notification center was a natural fit.

## One slot, a whole stack of buttons

A `Scaffold` gives you a single `floatingActionButton`. You are not limited to one
button, though — the slot takes any widget, so you stack them in a `Column` that
grows upward from the bottom-right corner:

```dart
floatingActionButton: Column(
  mainAxisSize: MainAxisSize.min,
  crossAxisAlignment: CrossAxisAlignment.end,
  children: [
    AppFab(
      heroTag: 'notif_clear_all',
      onPressed: _items.isNotEmpty ? _confirmClearAll : null,
      tooltip: l10n.clearAll,
      icon: const Icon(Icons.delete_sweep_outlined),
    ),
    AppFab(
      heroTag: 'notif_mark_all_read',
      onPressed: _items.any((n) => !n.isRead) ? _markAllRead : null,
      tooltip: l10n.markAllRead,
      icon: const Icon(Icons.done_all),
    ),
    AppFab(
      heroTag: 'notif_back',
      onPressed: () => Navigator.of(context).pop(),
      tooltip: MaterialLocalizations.of(context).backButtonTooltip,
      icon: const Icon(Icons.arrow_back),
    ),
  ],
),
```

Two deliberate choices there:

- **Nothing moves.** All three buttons are always mounted, so the stack's layout
  is fixed. When an action has nothing to act on — clear all on an empty list,
  mark all read when everything is already read — it stays put and greys out
  instead of vanishing. A button that comes and goes makes the one below it
  jump, and that motion is exactly what makes a UI feel unstable.
- **The list gets out of the way.** The FAB stack floats over the content, so the
  list carries bottom padding equal to roughly its height. Without it, the last
  notification sits under the buttons.

`AppFab` is our shared wrapper around `FloatingActionButton` — a circular outline
in the brand colour over the surface, with a brand-coloured icon. Routing every
FAB through it is what keeps the app from drifting into a mix of Material's tonal
default (which reads as a rounded square in a different colour) and the house
style.

Passing `null` for `onPressed` is the idiomatic way to disable a Material button,
and it does make the FAB non-tappable — but it does **not** grey it out.
`FloatingActionButton` only falls back to its disabled colours when you leave
`backgroundColor` and `foregroundColor` unset; set both explicitly, as the house
style does, and a disabled FAB keeps the full brand colour — looking enabled while
doing nothing. So `AppFab` reads `onPressed == null` itself and swaps to muted
surface-variant greys for the outline and the icon:

```dart
final enabled = onPressed != null;
final accent = enabled ? colors.primary : colors.onSurfaceVariant;
final border = enabled ? colors.primary : colors.outlineVariant;
```

## Gotcha #1: two FABs on a route trip the hero-tag check

This one is not obvious until it breaks.

Every `FloatingActionButton` participates in Flutter's [Hero](https://api.flutter.dev/flutter/material/Hero-class.html)
machinery for route transitions. When you don't give a FAB a `heroTag`, it gets a
**single shared default tag** — the same constant for *every* untagged FAB in the
app. With one FAB per screen that's fine. Put three on one route, and you have
three heroes claiming the same tag: Flutter throws a duplicate-tag assertion, and
even when it doesn't, route transitions animate the wrong button across the
screen.

The rule we landed on is boring and effective: **every FAB gets an explicit,
unique `heroTag`.** Note that "unique" has to hold beyond this route, too — the
Friends page's own FABs are still mounted underneath while the notification center
pushes on top of them, so the tags must not collide across both screens.

## Gotcha #2: making an icon-only button say what it is

An icon-only `FloatingActionButton` has no text for a screen reader and no label
for a long-press tooltip, so it should always carry one. For the two app actions
we had localized strings already. For **back**, we didn't — and it turns out we
never needed to add one:

```dart
tooltip: MaterialLocalizations.of(context).backButtonTooltip,
```

`MaterialLocalizations` ships a translated `backButtonTooltip` for every locale
Flutter supports, which is exactly the string a real `AppBar` back button would
use. Borrowing it keeps one source of truth for "Back" across the whole app
instead of a fifth translation that can drift.

## The result

The notification center is now a clean list with three outlined circles parked in
thumb range — **back**, **mark all read**, and above it **clear all** — that grey
out whenever there is nothing for them to do. The status bar stays the page's
colour, back works from the bottom-right the way it does on the full-screen map,
and the screen finally looks like it belongs to the same app as everything around
it.

It's a small change. But "the app bar can just be, well, not there" is one of
those realisations that quietly raises the floor on every screen you touch
afterwards.

If you want the surrounding context, the Friendship flow and the
[connection status work](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-12-small-polish-connection-status-and-call-end-card.html)
we shipped just before this are both written up on the blog, and there's a
[full overview of GwinG](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html)
if you're new here.

## FAQ

**Does removing the app bar break the system back button?**
No. The system back gesture and button are handled by the route, not the `AppBar`.
Removing the toolbar only removes the *on-screen* affordance — which is exactly
why we added the back FAB. Both work, and Android's predictive back is unaffected.

**Why not just use a `BottomAppBar` or `PersistentFooterButtons`?**
Either works. We chose a FAB stack because the app already has an established FAB
style on the Friends page, and reusing it made the notification center match
without introducing a fourth bottom-bar pattern to keep consistent.

**Can you have more than one `FloatingActionButton`?**
Yes — the `Scaffold` takes one `floatingActionButton` widget, but that widget can
be a `Column`, a `Row` or any layout containing several buttons. Just give each a
unique `heroTag`.

**Does the zero-height `AppBar` still push the body down?**
It reserves the status-bar inset (so content doesn't slide under the clock) but no
toolbar height, so the body starts right below the system status bar. We still
wrap the body in `SafeArea(top: true)` for the remaining insets.

— the GwinG team
