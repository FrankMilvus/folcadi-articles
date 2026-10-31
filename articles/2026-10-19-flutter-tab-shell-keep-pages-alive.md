---
layout: post
title: A Flutter tab shell that keeps every page alive
description: How GwinG hosts four tabs in one Flutter route with IndexedStack, preserves state across switches, and layers swipes without breaking inner gestures.
tags: showdev, flutter, mobile, ux
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-19-flutter-tab-shell-keep-pages-alive.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/flutter-tab-shell-keep-pages-alive.png
---

## The spinner-on-every-tab problem

A bottom navigation bar looks like a solved problem. Four destinations, a selected index, a page for each one. The naive implementation is a `pushReplacementNamed` on every tap: each destination is its own route, and switching tabs swaps the route.

That works until the app has state worth keeping. In GwinG, each main tab owns real, expensive state:

- Search runs a debounced user query, holds a map preview and polls presence for the visible results.
- Friends owns the chat list, chat tabs, scroll positions, an avatar cache and a WebSocket subscription.
- More is a menu whose keyboard cursor should survive a detour to another tab.
- Profile renders the signed-in user's data and media.

With route replacement, every tap re-ran `initState`: another friends request, another version check, another avatar prewarm, another presence poll, and a scroll position reset to the top. The app was not slow because the network was slow. It was slow because it kept throwing work away.

The fix was to host all four tabs in **one route** and switch the visible child instead of replacing the route.

## The shell in one widget

`HomeShell` is a `StatefulWidget` that takes an initial index, builds the four pages once, and keeps them in an `IndexedStack`:

```dart
Scaffold(
  body: FadeTransition(
    opacity: _tabFade,
    child: IndexedStack(
      index: _index,
      children: [
        for (var i = 0; i < _pages.length; i++)
          HeroMode(
            enabled: i == _index,
            child: _visited.contains(i)
                ? _pages[i]
                : const SizedBox.shrink(),
          ),
      ],
    ),
  ),
  bottomNavigationBar: const NavbarWidget(),
)
```

Three deliberate choices are packed into that tree.

### 1. Lazy visits, eager Friends

An `IndexedStack` builds every child it is given, even the ones that are not visible. Building all four tabs at startup would start four network paths before the user has asked for any of them. The shell therefore tracks a `_visited` set and substitutes an empty placeholder for tabs the user has never opened. A page's `initState` — and its first request — runs on the first visit, not at app launch.

The exception is Friends. Friends is the home tab in the product's mental model, so it is added to `_visited` at startup. By the time the user taps it, the list is already loaded or already loading. The tradeoff is one request at launch in exchange for never showing a spinner on a tab people open constantly.

### 2. One source of truth for the selected tab

The navbar does not navigate. It publishes an index to `selectedNavBarNotifier`; the shell listens and moves the stack. The same notifier is written by:

- a navbar tap,
- a horizontal page swipe,
- the keyboard tab shortcuts,
- a deep link or notification that asks for a specific tab.

Because every entry point writes the same value, the highlight, the visible page and the persisted "last tab" cannot disagree. A second notifier, `selectedPageNotifier`, records the tab for the next cold start. The shell renders; the notifiers decide.

### 3. A fade, not a rebuild

Switching tabs is a `FadeTransition` driven by a short `AnimationController`. It is subtle on purpose: the pages are already alive, so a large transition would draw attention to the fact that nothing was rebuilt. The duration also goes through the app's motion settings, so reduced-motion users get an instant switch.

## What "alive" actually buys

Keeping a page mounted is not just about avoiding a spinner. It preserves:

- **Scroll position** in long chat and search lists.
- **In-flight work**: a search request or a friend refresh that started before the switch can finish and update the tab while it is hidden.
- **Subscriptions**: WebSocket listeners and presence polling keep running, so returning to Friends shows current data instead of a stale snapshot.
- **Draft UI state**: an open search box, an expanded menu or a keyboard cursor is still where it was.
- **Image cache warmth**: avatars and media already decoded do not flash back in.

The price is that hidden pages are still *running*. That is the part the shell has to manage carefully.

## Hidden pages are still listening

Every page that registers a keyboard handler, a WebSocket listener or a timer is still registered while it is off-screen. Without a gate, pressing a search shortcut on the Profile tab could move a cursor in a Search page nobody can see.

The codebase uses two gates consistently:

```dart
// In a page's raw key handler:
if (ModalRoute.of(context)?.isCurrent != true) return false;
if (selectedNavBarNotifier.value != _myTabIndex) return false;
```

The first line rejects input while a pushed page or dialog is on top. The second rejects input while another tab is visible. The same predicate is passed to the F1 hotkey-help registry, so the cheat sheet only advertises chords that actually work on the current screen.

Hero animations need the same treatment. Off-screen tabs stay mounted, so their `Hero` widgets are still in the tree. Without `HeroMode(enabled: i == _index)`, a profile opened from Search could fly its avatar toward a hidden Friends row that happens to share the same tag. The shell enables heroes only for the visible child.

## Layering swipe navigation on top

Once the shell exists, adding a horizontal page swipe is straightforward — provided each page decides who owns the gesture.

### More and Search: a page-level wrapper

These pages have no inner pager, so they put `HomeTabSwipe` around the whole body. The wrapper recognises a decisive horizontal drag, applies distance and velocity thresholds, ignores drags near the left edge (the platform back gesture), and calls the shared tab-switch function with `-1` or `+1`.

Inner gestures keep priority automatically. On Search, the map preview pans horizontally, and the vertical scroll view owns vertical drags; the page wrapper only sees drags that neither of them claimed. That is why the shell-swipe test starts its fling *below* the map instead of on it.

### Friends: hand off at the edges only

Friends cannot use a page-level wrapper for normal swipes, because its body already contains a `PageView` for chat tabs. Swiping inside that pager must move between chat tabs, not between shell tabs.

The solution is a raw pointer listener around the `PageView`:

- It records the pointer position and the current page on pointer-down.
- On pointer-up it measures horizontal travel and rejects the gesture if it is mostly vertical.
- Only when the pager is already on its **first** tab and the swipe goes backward, or on its **last** tab and the swipe goes forward, does it hand off to the shell.

Because the listener observes raw pointer events and never joins the gesture arena, the inner pager keeps its normal drag behaviour. The shell only takes over at the boundary, which is exactly what a user expects: swipe through your chat tabs, and keep swiping to leave the page.

The shell swipe wrapper is still used on Friends — but only in the loading state, where no `PageView` exists yet. That detail is the subject of the companion post, [Flutter swipe navigation that keeps working while the app loads](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-19-swipe-navigation-during-loading.html).

### Not every page needs a swipe

The wrapper is opted into per page. Profile, for example, does not install a page-level swipe; its content is a vertically scrolling profile view, and the navbar and keyboard remain the navigation paths. This is a feature of the design, not an omission: gestures should be added where they do not compete with the page's own interactions.

## Pull-to-refresh and maps in the same tree

Search combines three gesture layers that could easily fight each other:

1. A horizontal page swipe at the shell level.
2. A vertical pull-to-refresh indicator.
3. A map that pans in both directions.

They coexist because each recogniser is scoped:

- The `RefreshIndicator` wraps a `SingleChildScrollView` with `AlwaysScrollableScrollPhysics`, so the pull works even when the result list is shorter than the viewport.
- The map owns drags that start inside its bounds.
- The page wrapper only accepts horizontal drags that the scroll view and map did not claim.

When a page adds a new horizontal child — a carousel, a swipeable row, a pager — the page-level swipe should be re-checked. The gesture arena will do the right thing most of the time, but "most of the time" is how dead zones and accidental tab switches get shipped.

## Back behaviour and persistence

The shell changes what "back" means. On a main tab, the system back button minimises the app; on a sub-page (Settings, a chat, About), it pops back to the shell. The root wrapper implements that distinction once, so individual pages do not need their own `PopScope` handlers.

Persistence has a subtle failure mode the shell had to fix. The last visible page index was being written even while the user was on a sub-page. After the process was killed and restored, the app could cold-start on a route that is not one of the four tabs and has no navbar. The restore path now normalises the saved index to a tab route before navigating; sub-page indices map to their parent tab. The rule is simple: **persist a destination, not an arbitrary route**.

## Testing a shell

The shell's tests are mostly about the invariants that are easy to break during a refactor:

- `home_tab_swipe_test.dart` checks the gesture thresholds, direction mapping, the edge guard and the desktop no-op.
- `more_shell_swipe_test.dart` and `search_shell_swipe_test.dart` assert that a swipe moves the shell notifier and the persisted page index.
- `friends_shell_swipe_test.dart` checks all three behaviours: an outward swipe at the first/last chat tab switches shell tabs, an inner swipe stays inside Friends, and a swipe during the loading skeleton switches shell tabs.
- Loading-state tests hold the async source open (a never-completing token read or friends fetch) so the dead state is actually reachable. A test that only pumps the loaded page cannot catch a missing wrapper.

The assertions are on navigation state, not on pixels. That keeps them stable across theme and layout changes while still failing when a gesture is silently dropped.

## Tradeoffs worth naming

**Memory versus refetching.** Every visited tab stays in memory. For four tabs with image-heavy lists this is a real cost, mitigated by lazy first visits and by disposing per-tab controllers only when the shell itself goes away. The alternative — rebuilding and refetching on every tap — is a worse user experience and more network traffic.

**Eager Friends.** Pre-building one tab at startup is a deliberate asymmetry. It spends a little launch work to remove a spinner from the tab users open most.

**Fade versus slide.** A slide transition suggests spatial movement, which promises that a swipe will move the page with your finger. The shell does not implement that kind of pager; it switches instantly (with a short fade) and treats swipes as commands. Promising less is more honest than a half-implemented parallax.

**One route, many states.** Because the tabs are not routes, deep links need a mapping layer that both navigates to the shell and selects the right tab. That is a small amount of code, paid once, instead of duplicating state ownership across routes.

## FAQ

**Why not use a `PageView` for the whole shell?**
A shell-level pager would own horizontal drags that several pages already use — chat tabs, maps, carousels and text selection. It would also keep every page's width in the layout and make state preservation depend on the pager's keep-alive behaviour. The `IndexedStack` plus per-page gesture wrappers is simpler to reason about.

**Do hidden tabs keep making requests?**
They can, and that is intentional for subscriptions and in-flight work. Each page still gates user input on visibility, and long-lived listeners are removed when the page is disposed.

**Why does Friends appear before I tap it?**
It is added to the shell's visited set at startup so it can load in the background. The other tabs stay unbuilt until first use.

**Does swiping work on desktop?**
No. The gesture wrapper is mobile-only; desktop uses the visible navigation bar and keyboard shortcuts. A horizontal mouse drag is usually selection, not navigation.

**What happens in right-to-left languages?**
The wrapper maps the swipe direction to the layout's text direction, and the destination is clamped at the first and last tab.

**How are dialogs and pushed pages handled?**
Both gates — "this route is current" and "this tab is visible" — fail while a pushed page or dialog is on top, so shell shortcuts and gestures stay out of the way.

## Where to go next

If you are building a shell like this, start with the state model: one notifier for the selected tab, one persisted index, and a lazy `IndexedStack`. Add gestures one page at a time, checking what each page already owns. Then write tests that assert the navigation state while the app is loading, not only after it has finished. The loading state is where navigation bugs hide.

For the specific loading-state fix that started this pair of posts, see [Flutter swipe navigation that keeps working while the app loads](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-19-swipe-navigation-during-loading.html). For what the app itself is, start with the [GwinG overview](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html).
