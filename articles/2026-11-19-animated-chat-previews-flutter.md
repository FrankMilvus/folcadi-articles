---
layout: post
title: "The chat list should move like the chat: animated message previews"
description: "Animated chat previews Flutter can afford: grapheme splitting, a four-emoji cap, prefix handling and TickerMode for kept-alive tabs."
tags: showdev, flutter, animation, performance
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-19-animated-chat-previews-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-19-animated-chat-previews-flutter.png
---

Animated chat previews Flutter draws in the chats list are the first thing a person sees after opening the app, yet for a long time ours were plain text. The conversation itself had motion — animated emoji in bubbles — while the list that summarised it stayed completely still. Fixing that turned out to be less about animation and more about parsing, prefixes and two tabs that quietly render the same rows. This is what the second pass taught us, and why the safest implementation is the one that refuses to animate too much.

## The list was the last static surface

The message bubble is the easy case. A message is either one emoji or it is not, so the bubble can make a single decision and swap one glyph for one animation.

A preview is different. It is a compressed line of text that must communicate a whole message in the width of a row. It can be a single emoji, an emoji in the middle of a sentence (`nice 🔥`), several emoji in a row, a sender prefix, or a draft. If the chat feels alive and the list looks like a database report, the mismatch is noticeable.

So the new rule is broader: every registered emoji in the preview animates, wherever it appears. Unregistered emoji and ordinary text stay exactly as they were. There is no special casing for the whole-message case, because the preview is not a message bubble and should not pretend to be one.

## The parsing problem behind animated chat previews Flutter renders

You cannot find emoji by splitting a string into code units. A single visible emoji can be several code points: a base character, a variation selector, a zero-width joiner, or a whole ZWJ sequence. Splitting naively gives you fragments, and fragments produce nonsense.

Dart's `characters` package gives the right unit: a grapheme cluster, which is what a reader perceives as one character. The preview walks the content grapheme by grapheme, buffering ordinary text and flushing it whenever a registered emoji appears:

```dart
for (final grapheme in content.characters) {
  final asset = animatedEmojiAssetFor(grapheme);
  if (asset == null || animated >= maxInlineEmoji) {
    buffer.write(grapheme);
    continue;
  }
  flushText();
  spans.add(
    WidgetSpan(
      alignment: PlaceholderAlignment.middle,
      child: Semantics(
        label: grapheme,
        child: AnimatedEmoji(asset: asset, size: 18),
      ),
    ),
  );
  animated++;
}
flushText();
```

The result is a `Text.rich` with a mix of `TextSpan` and `WidgetSpan` children. Text keeps the row's style; an emoji becomes a tiny square animation. If nothing in the content matches the registry, the widget returns the original plain `Text` with its one-line ellipsis, so the common case pays no new cost.

The grapheme walk is also why variation selectors behave. A heart that arrives with U+FE0F and a heart without it both normalise to the same registry entry, and neither leaves a stray invisible character in the text buffer.

## Four emoji, then stop

A preview is one line, and one line can hold a dozen emoji. Each animated emoji is a separate looping Lottie widget with its own ticker and its own composition. Ten of them in one row is not a richer preview; it is a battery test.

The cap is four animated emoji per preview. The fifth and later stay as text glyphs, even if they are registered. That is deliberately not "animate the first four and drop the rest": the rest remain visible, just static, so the message still reads correctly.

Four is a judgment call rather than a measured optimum. It is enough to convey a reaction burst, and low enough that a single row cannot spawn an unbounded number of animations. The important part is that the bound exists in the widget rather than in a comment asking future contributors to be careful.

## Prefixes are not message content

The preview line often starts with a name. Getting that prefix right is most of the work, because it differs by chat type.

For a direct message, an incoming message shows the content alone. An outgoing message keeps the localised `Me: ` prefix, so the list distinguishes "them" from "you" without an avatar label. The prefix stays a plain text span; only the part after it is inspected for emoji. That means a person named `Me` cannot accidentally animate, and a prefix ending in an emoji-looking character is still just text.

Groups are a little more involved. A group preview comes in as `Alice: hello`, so the tile splits at the first colon and space. If the last message was mine, the sender name is replaced with the localised "me" label; otherwise the original name is kept. Either way, the prefix is rebuilt as a text span and the content is passed to the animation parser:

```dart
final colon = preview.indexOf(': ');
if (colon == -1) return (prefix: '', content: preview);
return (
  prefix: isMe ? '${l10n.me}: ' : preview.substring(0, colon + 2),
  content: preview.substring(colon + 2),
);
```

The same widget is used by the friend tile and the group tile, so there is one implementation of this split and one place to fix it.

## One switch, not two

The obvious next question is whether the preview needs its own setting. It does not.

The existing switch is called **Animate emoji-only messages**, and its subtitle now mentions the chat-list preview too. The reasoning is simple: the preview *is* the message, compressed. A person who turned animations off because motion bothers them does not want the list to keep moving; a person who likes the effect does not want to discover a second toggle to make the list match the chat.

The preview also respects the app-wide motion level. If motion is off, the preview renders the plain text path, even if the emoji switch is on. One preference, two surfaces, no contradictory states.

## The trap: kept-alive tabs

This is the bug worth publishing, because it is invisible in a screenshot and obvious in a profiler.

The chats screen keeps its tabs alive so that switching between All and Friends does not rebuild the lists or lose scroll position. Keeping them alive means both tabs can hold the same rows at the same time. Each row contains an animated preview. When one tab is hidden, its widgets are still mounted, and mounted Lottie widgets keep ticking unless something stops them.

The fix is one wrapper around the page body. The tab page already used `HeroMode` to disable hero animations on the inactive page; `TickerMode` does the same job for every animation ticker:

```dart
final active = activeId == widget.tabId;
return HeroMode(
  enabled: active,
  child: TickerMode(enabled: active, child: child!),
);
```

`TickerMode(enabled: false)` pauses the tickers in that subtree without disposing the widgets. The hidden tab keeps its state, its scroll offset and its parsed spans, but it stops asking for frames. When the tab becomes visible again, the animations resume in place.

Without this, the list would animate twice for every visible row, and the extra work would be spent on pixels nobody can see. It is the kind of cost that does not show up until a low-end phone gets warm.

## Keeping the row short

A chat row is a tight horizontal budget: a 48 px avatar, a name, a timestamp, a status icon, an unread badge, and whatever width is left for the preview. The preview must not make the row taller, and the animation must not push the text around.

Two details handle that. The row height is still driven by the 48 px avatar, so the preview can grow from 14 px text to an 18 px emoji box without changing the tile's height. And the inline `WidgetSpan` keeps its intrinsic size inside the `Expanded` width. A stretched animation box would make Lottie centre its artwork somewhere in the middle of the row; an intrinsic box keeps the flame aligned with the text around it.

That second point is pinned by a regression test: an unprefixed emoji at the start of the row must be exactly 18 px wide at x = 0. It is a small assertion, but it protects the visual contract that makes the feature look intentional rather than merely enabled.

## Accessibility in a 14 px line

Each animated glyph is wrapped in `Semantics(label: grapheme)`, for the same reason as in the message bubble: Lottie has no text. A screen reader hears the emoji it would have heard from the plain string, and the visual reader sees motion.

The preview keeps its single line and ellipsis, and the semantics tree keeps the original characters. Neither the parser nor the animation is allowed to change what the message says.

## What the tests cover

The preview tests are behaviour tests, not snapshot tests:

- a single registered emoji animates, with and without a `Me: ` prefix;
- a registered emoji animates inside text, and multiple emoji animate in one preview;
- the four-emoji cap holds for a long run;
- the static path is used when the setting is off, when motion is off, and for an unregistered emoji;
- an unprefixed emoji stays 18 px wide at the start of the row.

The group tile gets its own test, because the sender-prefix split is the part most likely to regress. It pumps a real tile with an `Alice: 🔥` preview and asserts that the prefix stays text while the fire animates.

## The takeaway

The preview feature reuses almost everything from the bubble feature: the registry, the lookup, the widget, the setting, the semantics wrapper. What it adds is the part that only a list can teach you.

- Parse by grapheme, not by code unit, or Unicode will find the cracks.
- Put a hard cap on how many animations one row can create.
- Treat prefixes as chrome and inspect only the content.
- Reuse the user's existing preference instead of multiplying switches.
- Remember that kept-alive tabs render twice; pause the hidden copy with `TickerMode`.
- Measure the row against its existing height, not against the animation's preferred size.

The chat list now moves like the chat, but the important change is not visual. It is that the list can show motion without becoming a second, hidden battery drain.

If you want the first half of this story, the write-up on [animating emoji-only messages](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-16-animated-emoji-messages-flutter.html) covers the registry, the licence registration and the exact-match rule. For another example of motion that follows intent rather than a timer, see [hiding the chrome while scrolling](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-11-hide-status-bar-while-scrolling-flutter.html).

## FAQ

**Why not animate every emoji in a long preview?** Because each one is a separate looping animation. Four keeps the effect readable and bounds the work per row; the remaining emoji stay visible as static glyphs.

**Does the preview use a different setting from the chat?** No. It follows **Animate emoji-only messages** and the app-wide motion level.

**What about drafts and typing indicators?** They keep their existing static text. The parser only sees message content, not the labels around it.

**Does this change the row height?** No. The 48 px avatar still drives the tile height, and the emoji box is 18 px inside the existing line.
