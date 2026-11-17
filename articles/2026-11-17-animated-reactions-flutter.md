---
layout: post
title: "Small reactions, real motion: animating reaction chips in Flutter"
description: "Animated reaction chips in Flutter: one Lottie registry at 18 px, lifetime handed to the lazy list, a synced switch, and what to measure first."
tags: showdev, flutter, animation, performance
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-17-animated-reactions-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-17-animated-reactions-flutter.png
---

Animated reaction chips in Flutter look like a small feature until you count them. A reaction is a 14 px glyph, a count and a translucent rounded background, tucked against the edge of a message bubble. Then you scroll a busy group chat and realise one screen can hold dozens of chips at once. Animating that surface is a completely different job from animating one large emoji in the middle of a bubble, and that difference shaped every decision here.

This is the decision record for making reactions move: why we reused the existing emoji animation registry instead of building a second system, why the chip renders at 18 px rather than the 48 px used for emoji messages, how the lazy message list became the lifetime manager, and what we would measure before enabling motion everywhere.

## Why animated reaction chips in Flutter are a different problem

An emoji-only message is a single, deliberate, large thing. It sits in the middle of a bubble, the reader's eye goes to it, and one Lottie composition running there is cheap because there is exactly one of it.

A reaction row is the opposite. It is a small horizontal strip of chips, each one a glyph plus a number. A single busy message can carry six or eight reactions. A screen of messages can carry dozens. On top of that, every chip already has its own small animation: when a reaction is added or removed, the chip pops by scaling from 0.6 to full size with an overshoot curve.

So the question was never "can Lottie run in a chip". It was "what is the smallest amount of animation that still feels alive, and who owns it when the chip leaves the screen".

## The obvious approach, and why we skipped it

The obvious approach is a reaction-specific animation system: a controller per chip, a lifecycle tied to the row, maybe an exclusive "only one animates at a time" rule. That means a second set of assets, a second emoji-to-artwork map, and a second fallback to keep in sync when the user turns motion off.

We already had an 85-entry registry and a widget. Reusing them made the feature a rendering change rather than a new subsystem, and it kept one rule in one place: an emoji either has a bundled animation or it does not.

## One registry for messages, previews and reactions

The registry is a plain map from emoji to bundled Lottie asset. Lookup goes through a normaliser that trims the string and drops variation selectors, so `❤️` and `❤` resolve to the same entry instead of one of them silently missing. Adding an emoji is a JSON file plus one line in the map, not a new code path. The same registry also drives the animated emoji in the chats-list footer.

The lookup returns null for anything unregistered, and null is a valid answer:

```dart
final asset = animatedEmojiAssetFor(emoji);
final animate =
    asset != null &&
    animatedReactionsNotifier.value &&
    motionLevelNotifier.value != MotionLevel.off;
```

Anyone can send any emoji from the reaction picker, so "no bundled animation" is common rather than exceptional. Those reactions keep the static glyph, the same one they had before. There is no placeholder, no shimmer, and no attempt to substitute a similar animation. A still chip next to a moving one looks intentional; a wrong-but-animated chip does not.

## 18 px is the whole size budget

The static glyph is 14 px. The animated chip renders the same emoji at 18 px, which is large enough to read the motion and small enough that the count still sits beside it without wrapping.

`AnimatedEmoji` keeps a square box and fits the composition inside it, so artwork with a different aspect ratio — the fire, for example — is letterboxed rather than stretched. It also has an `errorBuilder` that returns an empty box of the same size. A corrupt or missing asset must not take the message bubble down with it; it should cost one chip's artwork and nothing else.

```dart
return Semantics(
  label: emoji,
  child: AnimatedEmoji(asset: asset, size: 18),
);
```

The size is deliberately not a preference. An 18 px glyph keeps the count beside it without wrapping; making it configurable would make the chip's height and the count's baseline user-dependent, for a decoration.

## The lazy list is the lifetime manager

A chat message list is lazy. It builds bubbles near the viewport and disposes them when they leave its cache extent. That means we did not need a visibility detector, an intersection observer, or a pause/resume policy for every chip. A reaction's animation lives exactly as long as its bubble does.

This is the single most important reason the feature is affordable. The animation is not "off while off-screen"; the widget that owns it no longer exists. What remains is the ordinary cost of a list item while it is on screen, plus one looping composition per animated chip inside it.

The caveat is that "off-screen" is not the same as "not built". A list keeps a small band of items alive beyond the viewport, so a few chips can still be animating just above or below the visible area. That is a fine default for a chat, but it is the first thing to check if you are porting this idea to a surface that keeps every row alive.

## Keep the key, keep the pop

Reactions are grouped by emoji, and each chip is keyed by the pair of emoji and count:

```dart
key: ValueKey('${e.key}:${e.value}'),
```

That key predates the animation, and keeping it was a deliberate constraint. When a count changes, the key changes, Flutter treats it as a new widget, and the existing pop replays. If we had keyed by emoji alone, a second person reacting would update the number without any acknowledgement, which is a worse interaction than a still chip.

The first build of a row disables the pop, so opening a chat does not cascade dozens of chips bouncing at once. Only a count change after that first build animates, and motion level off collapses the pop duration to zero.

The animated glyph sits inside that same pop wrapper, so the two motions compose instead of fighting. Adding a reaction scales the whole chip, animation included, and the loop continues underneath.

## Toggling the switch must not restart the row

The row listens to two values: whether reaction animation is on, and the app-wide motion level. Both are read inside one `AnimatedBuilder`, so changing either rebuilds the mounted chips in place.

A naive implementation puts the setting check above the whole row and keys the row by it, so toggling destroys and recreates every chip. Every pop replays, and a settings change looks like a burst of activity in a chat you are not looking at.

Listening lower down swaps only the glyph widget. The chip keeps its key, its scale animation keeps its state, and the count does not pop. The switch feels like a preference, not an event.

## An independent switch, synced like the rest

**Animate reaction emoji** lives in Settings → Appearance → Animated emoji, next to **Animate emoji-only messages**. The two are independent, and both default to on.

Independent matters because the costs are different. Emoji messages are rare and large; reaction chips are frequent and small. Someone may like one big animated emoji and find a wall of moving chips distracting, or the reverse. One switch for both would force a choice between two unrelated surfaces.

The value is stored under its own settings key and travels through the same settings map as every other appearance option, so it follows the account across devices.

## The first frame after a restart

A synced setting has a first-frame problem. The app can render the chats tab before the profile request returns, and for that moment it does not know whether reactions should animate. If the default is "on" and the stored value is "off", the first frame shows motion and the next frame removes it: a flash on every launch.

The fix is the same one the other appearance options use. The value is cached locally on change and read at startup, before the profile arrives, so the first frame already matches the last choice. The server sync then confirms or corrects it.

## Motion off means a still glyph, not a slower loop

The app has a master motion level. When it is off, reaction chips render the static emoji, exactly like emoji messages and the chat-list preview. We do not slow the loop down, shorten it, or run it once. Reduced motion is not a hint to animate more gently; for some people it is the difference between usable and not.

The same check covers both the glyph and the pop duration, so a motion-off chat has no looping artwork and no overshoot scaling. The feature disappears cleanly.

## Lottie carries no text

An animated composition is pixels. A screen reader gets nothing from it: no emoji name, no indication that the chip is a reaction at all. The count beside it may be the only text announced, which turns "two people reacted with a thumbs up" into "2".

Wrapping the animated glyph in `Semantics(label: emoji)` gives assistive technology the same word the static glyph would have exposed. The label is the emoji character itself, not a description like "thumbs up", because that is what the rest of the interface uses and what the reader expects to hear.

The static path needs no extra wrapper: a `Text` widget already carries its string. Only the animated path has to add meaning back.

## What I would measure before enabling animation everywhere

Reaction chips are a bounded surface: the list decides how many exist, and disposal is automatic. That is why we were comfortable turning them on by default. Before extending the same treatment to a grid, a pinned summary, or any list that keeps its children alive, I would want numbers for:

- **Live tickers.** How many Lottie animations are running on a reaction-heavy screen, including the cache-extent band above and below the viewport.
- **Frame time while scrolling.** Build and raster cost per frame on a mid-range Android device, with the same conversation measured before and after.
- **Decode cost at entry.** What happens when a chat opens and a screenful of compositions is decoded at once, rather than one large emoji at a time.
- **Memory and battery.** The retained cost of the decoded animations, and what an hour of reading does to the battery.

The principle is simple: animation is affordable when the widget tree bounds the number of animations. When it does not, the number is the feature's real budget, and it should be measured rather than assumed.

## The short version

- A reaction chip is a different performance problem from one big message emoji: many small loops instead of one large one.
- Reuse the existing emoji registry and `AnimatedEmoji`; do not build a second animation system.
- Render at 18 px, keep the count beside it, and leave unregistered emoji static.
- Let the lazy message list own the animation's lifetime. Check the cache-extent band before reusing the idea on a non-lazy surface.
- Keep the `emoji:count` key so the existing pop still fires when the count changes.
- Store the switch as its own synced setting, with a local first-frame cache so the row does not flash on launch.
- Motion off means the static glyph, in full.
- Give the animated glyph a `Semantics` label; Lottie has no text.

## FAQ

**Does this run in encrypted chats?** Reaction rendering is presentation only. It does not change what is stored or how anything is decrypted.

**What happens if an animation asset is missing?** `AnimatedEmoji` returns an empty box of the same size instead of throwing, so the layout holds and only that chip's artwork is missing.
