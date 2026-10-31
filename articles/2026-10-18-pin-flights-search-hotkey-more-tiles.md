---
layout: post
title: Pin flights, a real find hotkey, and colored More tiles
description: Three small GwinG polish changes — pin flights, a browser-standard find chord, and accent colors on the More tiles.
tags: showdev, flutter, ux, opensource
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-18-pin-flights-search-hotkey-more-tiles.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/pin-flights-search-hotkey-more-tiles.png
---

## Small changes you feel before you name them

Most of the work in a messenger is invisible: delivery state, encryption, reconnects, the thing that happens when the network drops halfway through a photo. Every now and then a batch of much smaller changes lands, and the app simply *feels* different. This is one of those batches.

Three things changed in GwinG this week. Pinning a chat or a message now has a short flight instead of a snap. The browser-standard find chord works on every search surface. And the three navigation tiles on the More tab picked up an accent color from the active theme.

None of them adds a feature you can point at in a changelog title. All of them remove a tiny moment of doubt.

## A pin should move, not teleport

When you pin a chat, the list reorders immediately. That is the correct data behaviour, but it used to be a hard cut: the row you were looking at vanished from one place and appeared in another, and your eye had to find it again.

The new pin flight keeps the row legible. Before the model reorders the list, the UI snapshots the selected row's rectangle. One frame later — after the reordered list has laid out — a ghost tile flies from the old rectangle to the row's new position in 320 ms. The real row is hidden at its destination while the ghost is in the air, so there is never a double image. The ghost's avatar hero is disabled so it cannot collide with the real row's hero.

The details are where the edge cases live:

- **The destination may be off-screen.** If the pinned section scrolled out of view, the ghost lands at the list edge — top for a pin, bottom for an unpin — and fades out. The pin still applies.
- **A second pin may arrive mid-flight.** The old ghost is cancelled before a new one starts.
- **The optimistic model update may not reach the expected state.** In that case the flight is skipped; the list simply reorders as before.
- **The same chat can be mounted in several kept-alive tab pages.** The row keys are namespaced per tab, so the flight finds the exact mounted row rather than a lookalike in another tab.

Pinned messages get the same treatment in the other direction. The message bubble measures a compact source rectangle, and the pinned bar is measured one frame after it mounts. A ghost — pin icon plus the same preview text — flies from the bubble to the bar in 340 ms and fades over its last fifth, revealing the real bar underneath. A missing source rectangle or a squeezed bar (the keyboard can do that) skips the flight; the pin still applies. Unpinning deliberately does not animate.

The whole thing is behind a new **Settings → Motion → Pin animation** toggle. It is on by default and follows the same rule as the other motion work: it only plays at the Full motion level. Reduced or Off means the old instant reorder, which is still perfectly usable.

### The entrance bug hiding in the same pass

While adding the flights we fixed a bug that had been making list entrances look wrong. The staggered entrance used a `SlideTransition` with an offset of `18`. `SlideTransition` treats that number as a **fraction of the child's size**, not a pixel count. Rows were starting roughly eighteen times their own height below their final position. The fix was to use a `Transform.translate` in logical pixels, so the rows now move 18 px.

The second half of the fix was about replays. Rows are disposed when they scroll far out of view, so a keyed entrance could replay when they came back. Each row now animates only on its first build for the lifetime of the Friends view. Rows revealed by keyboard navigation still skip the entrance on purpose.

## The find chord should mean find

`main+F` is the browser-standard find chord. On GwinG it is `Alt+F` by default, or `Ctrl+F` if you choose the alternative in Settings → Shortcuts. Until now it only worked in some places, which made it feel unreliable: a shortcut that works *sometimes* is worse than no shortcut, because you stop trusting it.

It now focuses the search field on every search surface:

| Surface | What happens |
|---|---|
| Friends chats list | Opens the in-place search box and focuses it. Pressing F again re-focuses the field and selects the current query. Escape still closes the box. |
| In-chat message search | Opens the search panel and focuses its field. While the panel is open, its own handler owns F so the query can be re-focused and selected. |
| Search tab | Alias of the existing focus chord, and F additionally selects the current query. |
| Contacts | Focuses the friend filter above the list. |
| Create Group → Members | Focuses the friend picker; inert on the other steps. |
| Group info → Add members | Re-focuses the friend picker in the add-members sheet. |

Two implementation details made the difference. On the chats list, the F check runs *before* the "the search field already has focus" early return, so the chord also works while you are typing. In the chat page, the search panel registers its own keyboard handler while it is mounted, because the chat page's shortcut scope is inactive when the panel is on top.

No new localization keys were needed. The help rows reuse the existing strings for "Search chats", "Search in chat", and "Focus the search box", so the cheat sheet stays consistent without translating the same idea twice.

## Color as a signpost

The More tab has three navigation tiles: Settings, My Space, and Switch account. They were all the same default icon color. Now Settings uses the `secondary` color, My Space uses `tertiary`, and Switch account uses `primary`.

The palette is resolved once from the active theme, so the tiles follow light/dark mode and the user's accent preset. There is no behaviour change, no new copy, and no server sync. It is a small piece of wayfinding: three destinations, three visual anchors, one less moment of reading the labels.

## FAQ

### Does pinning still work if motion is disabled?

Yes. The data change is independent of the animation. With motion off, the row or message simply moves to its new place without the ghost flight.

### Does `Alt+F` work while I am typing in the search box?

Yes. On the chats list the chord is checked before the focus check, and it selects the current query so you can replace it immediately.

### Do the More tile colors change with the theme?

Yes. They are resolved from the active theme palette, so they follow the light/dark mode and accent preset rather than being fixed colors.

## Why this batch matters

A language-exchange app is a place where people already have to work a little: find the right person, switch languages, read a translation, check a pronunciation. The interface should not add its own small uncertainties on top of that.

A pin that flies is not a feature. A find chord that works everywhere is not a feature. A colored tile is not a feature. But together they make the app feel like it is paying attention — and that is the point of polish work.

If you want the implementation-level companion to this piece, the [shared media gallery](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-shared-media-gallery-in-flutter.html) and [photo albums](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-photo-albums-in-flutter.html) write-ups cover the same "small detail, real feel" territory.
