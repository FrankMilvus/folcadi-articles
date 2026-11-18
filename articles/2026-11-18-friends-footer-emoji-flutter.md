---
layout: post
title: "Turning dead scroll padding into a little animated fire"
description: "How a blank strip reserved for floating buttons became an animated footer emoji in Flutter: sliver geometry, a synced picker, and a still fallback."
tags: showdev, flutter, ux, animation
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-18-friends-footer-emoji-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-18-friends-footer-emoji-flutter.png
---

Every chat list ends somewhere, and ours used to end in nothing. The last conversation row sat above a blank strip of screen, and that strip existed for a practical reason: three floating action buttons live in the bottom-right corner, and the last row had to scroll clear of them. The padding was correct. It was also dead space that every user paid for on every short list.

Now an animated footer emoji sits in that space — a little fire by default. This is the decision record for turning reserved padding into a feature without changing the scroll geometry, and for keeping an animated footer emoji in Flutter cheap enough to leave on.

## The blank strip we were already paying for

The chats list reserves room for the tallest stack of floating buttons: search, the extra button some accounts get, and contacts. A mini floating button occupies 48 dp vertically, gaps sit between them, and the scaffold adds its own margin above the safe area. The result is roughly 184 px of reserved height at the end of the list.

That number is not decoration. It is the reason the last row can be tapped instead of sitting under a button. It is also why the empty state and the populated list share the same bottom padding: the geometry should not change just because the content did.

So the constraint was fixed before the feature existed. We were not looking for space to add; we were looking at space we had already bought and never used.

## Do not add a footer below the list

The tempting implementation is a column: the list, then a footer underneath it. It looks harmless, and it quietly changes two things.

First, it adds scroll length. The list now ends later than it used to, so the last row travels further before it settles. Second, it makes the geometry depend on whether the footer is enabled. Turn the decoration off and the list gets shorter; turn it on and it grows. A preference should not resize the conversation list.

The version we shipped puts the footer inside the scroll view instead, as the last sliver. The list's total extent stays the same whether the emoji is on, off, or replaced by an empty box of the same size.

## SliverFillRemaining is margin-top: auto

The footer is a `SliverFillRemaining` with `hasScrollBody: false`. On a list shorter than the viewport, it receives all the leftover height, which pushes its child to the real bottom. In CSS terms it is `margin-top: auto`; in a chat list it means a two-row list has one enormous gap between the second row and the emoji, rather than a small decoration floating in the middle of the screen.

On a long list there is no leftover height, so the sliver falls back to a minimum. That minimum is the height of the button stack, which preserves the original reason the space existed: the last row still clears the buttons.

```dart
return SliverFillRemaining(
  hasScrollBody: false,
  child: ConstrainedBox(
    constraints: BoxConstraints(minHeight: AppFab.stackHeight(3)),
    child: Align(
      alignment: Alignment.bottomCenter,
      child: FriendsFooterEmoji(scrollController: controller),
    ),
  ),
);
```

The `Align` matters as much as the sliver. Without it, the emoji would stretch or sit at the top of the expanded area. With it, the unused height lives above the emoji and the decoration stays pinned to the bottom edge.

## Same geometry with the switch off

When the footer is disabled, the widget still renders a square of its normal size with nothing inside. It does not return `SizedBox.shrink()`, because collapsing the box would remove height from the sliver and shorten the list. The switch would then change scroll behaviour as well as appearance.

That is the rule in one sentence: the setting decides what is painted, never how far the list scrolls.

It also makes the toggle feel instant. There is no relayout of the rows, no jump in the scroll position, and no re-measure of the viewport: one box goes from containing an animation to containing nothing.

## An animated footer emoji in Flutter, centred clear of the buttons

The floating buttons are right-aligned, so the footer is centred horizontally. That keeps it visually clear of the stack without needing to know the width of a button or the exact device inset.

An 8 px gap plus the device's bottom inset separates the emoji from the screen edge. The list's own bottom padding already accounts for the dock and the button stack, so the footer is not compensating for them again; it is only adding breathing room.

The result is a decoration that looks placed rather than dropped. On a wide desktop window it stays centred while the buttons remain at the far right.

## Reuse the registry, do not build a second one

The emoji comes from the same 85-emoji bundled animation registry used by emoji-only messages. That gives the picker one set of choices, the renderer one lookup, and the fallback one rule: if the emoji has no bundled animation, draw the glyph.

The picker only offers emoji that do have an animation, so the chosen footer always moves unless the app-wide motion level is off. A stored value that is not in the registry is ignored rather than rendered as a mystery glyph, which protects against an old or hand-edited setting.

Reusing the registry also means the footer inherits variation-selector normalisation and the same asset-loading behaviour as the rest of the app. There is no footer-specific asset pipeline to keep alive, and no second list of emoji to drift out of date. The same set also powers [animated reaction chips](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-17-animated-reactions-flutter.html).

## Glyphs in the grid, motion in the preview

The picker shows the full bundled set. Playing all of those Lottie compositions at once would be absurd: the sheet would ask the device to decode and loop dozens of animations for a choice the user has not made yet.

So the grid renders plain text glyphs. One animated preview at the top shows the current selection. Tapping an emoji updates the preview only; the sheet stays open, and nothing is saved until the user presses Apply.

That extra step is deliberate. An animation is a poor thing to judge from a still frame, and a footer is something the user will see every day. Previewing first and committing second means the choice is made after seeing the motion.

## Two settings that follow the account

The feature stores two values: which emoji, and whether the footer is enabled. Both travel through the same settings map as the other appearance options. There is no new endpoint and no new persistence format.

Because the whole settings map is pushed together, the footer choice follows the account across devices exactly like the theme, the chat background or the animation switches. Sign in on a new device and the same emoji is waiting at the bottom of the list.

An unknown emoji in the stored settings is ignored. That keeps the renderer's promise simple: the footer either shows something from the registry or shows the default, never a missing asset.

## The first frame after a restart

Like the other appearance settings, the footer values are cached locally as well as synced. The cache is read at startup, before the profile request returns, so the first frame already has the right emoji and the right switch state.

Without that, a list could render with the default fire, then swap to the chosen emoji a moment later. On a slow connection the flash could last long enough to notice, and a decoration that flickers on every launch is worse than no decoration at all.

The cache is a bridge, not a second source of truth. The server value still wins once it arrives.

## Motion off means a still glyph

The master motion level is respected. When it is off, the footer renders the emoji as text at a proportional size instead of playing the animation. No loop, no ticker, no exception for decoration.

That mirrors the emoji-message fallback, so the whole app has one answer to "what happens when someone turns motion off": the content stays, the movement does not.

## Decoration should be silent

The footer is wrapped in `ExcludeSemantics`. It carries no information: it is not a button, it does not show a count, and it does not indicate state. A screen reader announcing "fire" at the end of a list of conversations would be noise dressed as content.

Excluding it also keeps the semantics tree honest. The list ends where the conversations end, and the decoration is invisible to assistive technology in the same way a background pattern is.

## The ticker budget at the bottom of a long list

A looping animation at the end of a long list is a small but permanent cost. If the user is reading the top of a hundred conversations, the footer is nowhere near the viewport, and its ticker has no reason to run.

The footer mutes its animation while the list is more than 400 px from the end. Scrolling back towards the bottom re-enables it before it comes into view, so the motion is already running by the time the user sees it. The check also tolerates the brief moment when one controller is attached to two scroll views during a tab or search swap; in that case it animates rather than asserting.

This is the same principle as the lazy message list: do not pay for what nobody can see.

## What it costs, and what it buys

The cost is one looping composition, muted when far from the bottom, plus a sliver that already had to reserve space. The scroll length is unchanged, and empty tabs show no footer at all.

What it buys is harder to measure: a list that feels finished at the bottom instead of stopping in a blank strip. The space was already reserved; the only new decision was to treat it as part of the design.

## The short version

- The chats list already reserved about 184 px so the floating buttons would not cover the last row.
- Put the footer inside the scroll view as a sliver; do not add a footer below the list.
- Use `SliverFillRemaining` with `hasScrollBody: false` so a short list gives all unused height to the footer.
- Keep a minimum height equal to the button stack so a long list still clears the buttons.
- Render an empty box of the same size when the switch is off, so the geometry never changes.
- Reuse the bundled emoji registry and keep the glyph fallback for anything unregistered.
- Show glyphs in the picker grid and animate only the preview; require Apply before saving.
- Sync the emoji and the switch through the existing settings map, with a local first-frame cache.
- Mute the ticker when the footer is far from the viewport, and exclude it from semantics.

## FAQ

**Why not put the emoji in the floating button stack?** Because the stack is interactive and right-aligned. The footer is decoration and belongs in the list's own space, where it can be centred and scrolls with the content.

**Does the footer appear on empty tabs?** No. Empty tabs keep their existing centred message and show no footer.

**What happens if the chosen emoji loses its animation?** The stored value is validated against the registry when applied and when read from the cache. An unknown value is ignored, and the default remains.

**Does the footer change the scroll position when it is toggled?** No. The box keeps its size and the sliver keeps its extent, so only the painted content changes.

**Is this safe in encrypted chats?** The footer is on the chats list, not inside a conversation. It is presentation only and does not touch message content.
