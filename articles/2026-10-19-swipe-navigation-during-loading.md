---
layout: post
title: Flutter swipe navigation that keeps working while the app loads
description: A swipe should not wait for the network. How GwinG keeps Flutter tab navigation alive through session loading, skeletons, and late data.
tags: showdev, flutter, ux, mobile
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-19-swipe-navigation-during-loading.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/swipe-navigation-during-loading.png
---

## The swipe that only worked after the list appeared

GwinG is a language-exchange messenger with four main tabs: More, Search, Friends and Profile. On a phone, a horizontal swipe on a page body moves to the neighbouring tab, just like tapping the bottom navigation bar. It is a small convenience, but it is the kind of gesture people use without thinking.

A reader reported that it did not always work. Opening the Search tab on a slow connection and swiping left did nothing. As soon as the user list appeared, the same swipe worked. The gesture was not broken. It was simply not there during the part of the session where the app was waiting for the network.

That report was a useful reminder: **a loading state is still a screen**. If a screen can be looked at, it can be swiped, tapped, or dismissed. Treating loading as a placeholder that does not need navigation logic means every slow connection briefly traps the user on one tab.

## Two tabs, three loading states

The first step was to stop thinking about "loading" as one state. Search alone has two of them:

1. **The sign-in spinner.** Before the app has finished checking its saved session, Search renders a full-page `CircularProgressIndicator` instead of the map, search field and results. This is a different widget tree from the loaded page.
2. **The results skeleton.** Once the session is ready, the normal page renders, but the results area shows shimmering skeleton rows while the first request is in flight.

Friends has a third:

3. **The chats skeleton.** The chat list shows skeleton rows until the first friends response arrives. Its shell hand-off lives inside a `PageView` that only exists once the list is loaded.

Only one of those three was originally wrapped in the swipe handler. The results skeleton was already inside the normal Search page body, so it inherited the gesture. The sign-in spinner was an early return with no wrapper. The Friends skeleton replaced the entire page that owned the `PageView`, so its swipe logic was gone too.

## The wrapper: one page, one horizontal gesture

The shell uses a small widget called `HomeTabSwipe`. It is deliberately boring:

```dart
GestureDetector(
  behavior: HitTestBehavior.opaque,
  onHorizontalDragStart: (details) {
    _dx = 0;
    _ignored = details.globalPosition.dx < _edgeGuard;
  },
  onHorizontalDragUpdate: (details) {
    if (!_ignored) _dx += details.delta.dx;
  },
  onHorizontalDragEnd: _onDragEnd,
  onHorizontalDragCancel: () => _dx = 0,
  child: widget.child,
)
```

A few details matter more than they look:

- **Opaque hit testing.** A swipe that starts on the gap between rows still counts.
- **A distance and a velocity threshold.** A slow drag needs 48 logical pixels; a quick flick needs 320 pixels per second. Direction comes from the flick when there is one, otherwise from the finger's travel.
- **An edge guard.** Drags starting within 24 pixels of the left edge are left to the platform's back-swipe gesture.
- **RTL awareness.** "Next tab" is left in a left-to-right layout and right in a right-to-left one.
- **Mobile only.** On desktop, tabs are switched with the keyboard or a click; inventing a drag gesture for a mouse would only fight text selection.

Inner horizontal gestures keep priority. A map pans, a horizontally scrolling list scrolls, a text field selects text, and an inner `PageView` pages. The page-level wrapper only sees the drags that no child claimed. That is exactly the behaviour a shell wants.

## Fix 1: make Search swipeable from the first frame

The sign-in branch returned its own `Scaffold`, so the fix was to wrap its body in the same gesture widget the loaded page uses:

```dart
if (_token.isEmpty) {
  return Scaffold(
    appBar: AppBar(toolbarHeight: 0, /* ... */),
    body: HomeTabSwipe(
      onSwipe: _onSwipeToShell,
      child: const Center(child: CircularProgressIndicator()),
    ),
  );
}
```

`_onSwipeToShell` is the same handler as before: clamp the destination to the four tabs, give a small haptic, and call the shared `selectNavTab` function so a swipe and a navbar tap can never drift apart.

The important part is not the extra widget. It is the rule behind it: **the swipe wrapper belongs at the page boundary, not around the success UI**. If the page can render a full-screen waiting state, that state goes inside the wrapper too.

## Fix 2: give the Friends skeleton the shell hand-off

Friends is more interesting because its swipes normally do two different jobs. Inside the loaded page, a `PageView` moves between chat tabs (All, Encrypted, Friends, and any custom tabs). Only when you reach the first or last chat tab and keep swiping does the shell take over and switch to Search or Profile.

That hand-off is implemented with a raw pointer listener around the `PageView`. It watches pointer-down and pointer-up, measures horizontal travel, rejects vertical drags, and hands off only at an outer edge. The listener is outside the gesture arena on purpose: the inner pager still owns normal tab swipes.

While the chats are loading, there are no chat tabs and no `PageView`. There is nothing to page through, so a horizontal swipe can go straight to the neighbouring shell tab:

```dart
child: _loading
    ? HomeTabSwipe(
        onSwipe: _handOffEdgeSwipe,
        child: const _FriendsSkeleton(),
      )
    : _loadFailed
        ? AppErrorState(/* ... */)
        : MotionEntrance(/* the loaded page */),
```

This reuses the same edge-hand-off function as the loaded page, including its clamping and haptic feedback. No new navigation path, no second definition of "which tab is next".

## Testing gestures across async boundaries

The regression tests are the part worth copying. A widget test that simply pumps a loaded page cannot catch this class of bug, because the dead state only exists while an async operation is pending.

For Search, the test holds the secure-storage read open. The method-channel mock returns a future that never completes, so the page stays on the sign-in spinner. The test asserts that the spinner is on screen, flings from the middle of the page, and checks the shell's tab notifier:

```dart
final tokenRead = Completer<Object?>();
// secure-storage mock returns tokenRead.future for every read

await tester.pumpWidget(/* Search inside MaterialApp */);
await tester.pump();
expect(find.byType(CircularProgressIndicator), findsOneWidget);

await tester.flingFrom(
  const Offset(400, 300),
  const Offset(-300, 0),
  800,
);
await tester.pump();
await tester.pump(const Duration(milliseconds: 300));

expect(selectedNavBarNotifier.value, 2); // Friends
```

For Friends, the test injects a `FriendsService` whose `fetchFriends()` never settles. It asserts that skeleton rows are visible, flings the `HomeTabSwipe`, and checks that the shell moved to Search.

Two practical notes:

- **Do not use `pumpAndSettle` while a shimmer is animating.** A repeating animation never settles. Explicit `pump` calls with a duration are deterministic and fast.
- **Make sure the test fails without the fix.** Asserting "some widget exists" is not enough; assert the navigation result. In both cases the missing wrapper leaves the tab notifier unchanged, so the test fails for the right reason.

We also verified the results-skeleton state separately, including drags that start directly on a skeleton row. It already worked, which narrowed the fix to the two dead states instead of adding wrappers everywhere.

## What this changes for users

Nothing about how the loaded pages look. The only visible difference is that a swipe now works while the app is still checking the session, the friends list, or the first page of search results. On a good connection the change is invisible; on a bad one it removes a two-to-ten-second dead zone.

## Lessons that generalise

1. **Audit every early return.** A `return Scaffold(...)` above the main layout is a separate screen, and it needs the same navigation affordances.
2. **Wrap at the page boundary.** Put page-level gestures around the whole body, not around the widget that renders data.
3. **Let inner gestures win.** Maps, pagers, and text selection should keep their drags; the shell only handles what is left.
4. **Test the waiting state.** Hold the async source open in a widget test. If the test cannot reach the state, it cannot protect it.
5. **Reuse the navigation function.** Swipe, tap and keyboard all call one `selectNavTab`; a loading-state wrapper should use the same callback, not reimplement the rules.

## FAQ

**Does wrapping a spinner in a gesture detector cost anything?**
No. It is one gesture recognizer on an otherwise static subtree. It does not rebuild during loading and does not delay the request.

**Why not make the whole shell a `PageView` and get swipes for free?**
Because several pages already own horizontal drags, and a shell-level pager would constantly compete with them. The architecture behind this choice is the subject of the companion post, [A Flutter tab shell that keeps every page alive](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-19-flutter-tab-shell-keep-pages-alive.html).

**Does this work in right-to-left languages?**
Yes. The wrapper maps "next" to the left in LTR and to the right in RTL, and the destination is still clamped at the first and last tab.

**Why only mobile?**
The gesture is a touch affordance. Desktop users have a visible navigation bar, keyboard shortcuts and a mouse; a horizontal mouse drag is usually selection, not navigation.

**What about the platform back gesture on iOS?**
Drags that start within 24 logical pixels of the left edge are ignored, so the system back swipe keeps working.

If you want the wider context, the [GwinG overview](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html) explains what the app is, and the shell post covers how the four tabs stay alive without reloading.
