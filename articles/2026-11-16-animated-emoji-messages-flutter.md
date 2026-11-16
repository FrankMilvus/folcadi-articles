---
layout: post
title: "Big emoji that move: animating emoji-only messages with Lottie"
description: "How animated emoji Flutter bubbles stay predictable: an exact-match registry, Lottie assets, variation-selector handling and a screen-reader label."
tags: showdev, flutter, lottie, ux
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-16-animated-emoji-messages-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-16-animated-emoji-messages-flutter.png
---

An animated emoji Flutter feature looks like a one-line change: find the emoji, play a Lottie file. Then you try it and the edge cases arrive. A message that is only emoji already renders as a 48 px glyph in our chat, so the animation has to replace that glyph exactly, not sit beside it. A fire followed by a fire should stay two static glyphs. A heart written with or without a variation selector should still resolve. And a screen reader has to be told what the picture means. This is the story of those constraints, and the small registry that keeps them honest.

## The rule: one emoji, exactly

Our chat already had a rule for emoji-only messages: if the trimmed text is nothing but emoji, render it as one large glyph instead of ordinary body text. That rule is deliberately strict. A message with a word in it is text. An emoji caption on an image is a caption. A reply with a reaction is a reply.

The animation slots into that same branch, which is why it stays so contained. The message widget takes an opt-in flag, and only the emoji-only branch passes it. When the flag is on, the widget asks whether the *entire* trimmed message matches a registered emoji. `🔥` plays. `🔥🔥` does not, and neither does `🔥 hi` or a photo with a flame caption. Those keep the static enlarged glyph they always had.

That single decision prevents most of the bugs you would otherwise chase. There is no partial replacement inside a sentence, no animation competing with link spans, and no special case for media. The animation is a presentation swap at the point where the message has already been classified.

## Designing the animated emoji Flutter registry

The implementation is a constant map from an emoji to an asset path:

```dart
const Map<String, String> kEmojiAnimations = {
  '👍': 'assets/animations/emoji_thumbs_up.json',
  '❤️': 'assets/animations/emoji_red_heart.json',
  '🔥': 'assets/animations/emoji_fire.json',
  '🎉': 'assets/animations/emoji_party_popper.json',
};
```

The real map has 85 entries, grouped by intent: the app's twenty default reactions, common faces, gestures, hearts, and a handful of celebration symbols. The grouping is for humans reading the file; the lookup does not care about categories.

The important property is that adding an animation is one line plus one JSON file. The assets directory is already declared in `pubspec.yaml`, so a new file needs no build configuration. The key must be the emoji exactly as a keyboard would produce it. That sounds obvious until you try to type a keycap sequence or a flag into a source file and discover that editors and input methods disagree about invisible characters.

There is no database, no network call and no generated code. The registry is data, the lookup is a map access, and the failure mode is a `null` that means "keep the glyph". That is the right size of abstraction for a decoration.

## Normalising what the keyboard actually sends

Unicode gives the same visible emoji more than one byte sequence. A heart can arrive as U+2764 alone or as U+2764 followed by U+FE0F, the variation selector that asks for emoji presentation. Both look like `❤️` in most interfaces. A map with one spelling would miss the other.

So the registry builds a second, normalised map and every lookup runs the same transformation:

```dart
String _normalizeEmoji(String text) =>
    text.trim().replaceAll('\uFE0F', '').replaceAll('\uFE0E', '');

final Map<String, String> _kNormalizedEmojiAnimations = {
  for (final entry in kEmojiAnimations.entries)
    _normalizeEmoji(entry.key): entry.value,
};

String? animatedEmojiAssetFor(String text) {
  final normalized = _normalizeEmoji(text);
  if (normalized.isEmpty) return null;
  return _kNormalizedEmojiAnimations[normalized];
}
```

Trimming is part of the same step. A message that is "just an emoji" often has a stray space at the end, and a static 48 px glyph tolerates that silently. The animation path should too.

Two details are worth calling out. The normalisation runs on both the registry keys and the incoming text, so one spelling in the source file covers every spelling a platform produces. And the function returns `null` for anything that is not an exact single match after normalisation, which keeps the caller's fallback trivially simple.

## Fitting Lottie into a text-sized hole

The static glyph is 48 px. The animation is a 72 px square. That is not an accident: the Lottie artwork has its own internal padding, so a box slightly larger than the old glyph gives the motion room without making the bubble jump.

The widget is small enough to read in one sitting:

```dart
class AnimatedEmoji extends StatelessWidget {
  const AnimatedEmoji({super.key, required this.asset, this.size = 72});

  final String asset;
  final double size;

  @override
  Widget build(BuildContext context) {
    return Lottie.asset(
      asset,
      width: size,
      height: size,
      fit: BoxFit.contain,
      errorBuilder: (context, error, stackTrace) =>
          SizedBox(width: size, height: size),
    );
  }
}
```

`BoxFit.contain` matters because the bundled set is not uniform. Most files are square: the Noto animations are 1024×1024 at 60 fps. The fire animation is 500×690 at 24 fps and loops. If the widget stretched everything to fill a square, the fire would distort; if it cropped, the flame would lose its tip. Containing the composition inside the box preserves both.

The `errorBuilder` is the least glamorous line and the one I would keep in any app. A missing or corrupt asset should leave an empty square, not take down the message bubble with a red error box. A decoration must never be able to break the content it decorates.

## The switch and the fallback

The setting lives under Appearance and is called **Animate emoji-only messages**. It defaults to on. Turning it off restores the exact static glyph, with no restart.

Two rules interact here. The first is the user's preference. The second is the app-wide motion level: when motion is set to off, the animation is skipped even if the emoji switch is on. That is the correct precedence. Someone who has asked the whole interface to stop moving should not have to find a second, narrower switch to stop one more thing.

The choice is stored with the rest of the appearance settings and cached locally, so the first frame after a restart already knows which path to take. Without that cache you get a visible flash: static glyph, then animation. Caching a boolean is cheap; a flashing bubble is not.

The build listens to both values and re-renders when either changes:

```dart
final animate = animatedEmojiNotifier.value &&
    motionLevelNotifier.value != MotionLevel.off;
```

No animation controller of our own, no timers, no manual lifecycle. Lottie owns the frames; we only decide whether the widget exists.

## A screen reader cannot watch a Lottie

An animation is pixels. It carries no text node, so without help a screen reader would encounter an unlabelled box where the message used to be. That is a regression disguised as a visual upgrade.

The fix is one wrapper around the animation:

```dart
Semantics(
  label: display,
  child: AnimatedEmoji(asset: animation),
)
```

The label is the emoji itself, which is what the static glyph exposed before. A screen-reader user hears the same character they would have heard from the text path, and the visual user sees motion. Neither experience is treated as the lesser one.

The same wrapper appears wherever the animation is reused, because accessibility is a property of the widget boundary rather than of one call site.

## Licensing animations that are not a package

Flutter's licence page collects licences from packages automatically. Bundled assets get no such treatment. If you ship an animation set and do nothing, About → Licenses will be silent about it, which is both inaccurate and impolite.

Our set is 85 animations. 84 are Google's Noto Animated Emoji under the SIL Open Font License 1.1; the remaining fire animation came with the project. The full licence text ships as `OFL.txt` beside the JSON files, and the app registers it explicitly:

```dart
void _registerEmojiAnimationLicense() {
  LicenseRegistry.addLicense(() async* {
    final license = await rootBundle.loadString('assets/animations/OFL.txt');
    yield LicenseEntryWithLineBreaks(const ['Noto Animated Emoji'], license);
  });
}
```

That is the entire integration: load the text from the asset bundle, yield it under a readable name, and Flutter folds it into the standard licence page. No package, no plugin, no build step.

The broader lesson is that "bundled asset" is a licensing category the tooling does not track for you. Fonts, sound effects, sample images and animation sets all need the same deliberate registration if you want the About screen to tell the truth.

## The five-megabyte question

The full set costs roughly 5.4 MiB of assets. That is real weight in an installer, and it is the kind of number that deserves a sentence rather than a shrug.

We accepted it because the animations are local, instant and offline. There is no request when a message arrives, no placeholder, no failure path for a flaky connection, and no privacy question about which emoji a person used. The set is also bounded: it is a curated list, not a download-on-demand catalogue that could grow without limit.

The trade-off is honest: we spent installer size on a decoration. If the set grew much larger, the calculation would change and lazy loading would start to look attractive. For 85 files, bundling keeps the code simple and the behaviour deterministic, which is worth more than a few megabytes on a desktop install.

## What the tests pin down

The tests split cleanly along the same boundary as the code.

The registry gets pure unit tests: exact matches, common emoji, variation-selector normalisation, whitespace trimming, and rejection of multi-emoji, mixed text and empty input. The widget gets behaviour tests: it plays when enabled, stays static when the setting is off, stays static when motion is off, and stays static when the caller does not opt in. The settings value also gets a round-trip test, because a preference that does not survive a restart is not really a preference.

The emoji, settings and message suites all pass. None of that replaces opening the app and looking at a heart, but it does mean the fallback paths are treated as first-class behaviour rather than as leftovers.

## What I would keep

If you are adding motion to a small, well-defined piece of content, the shape of this feature is reusable:

- Put the decision in one predicate and make everything else a consumer of it.
- Keep a single source of truth for the mapping, and make the fallback the boring default.
- Normalise the input before you compare it, especially when the input is Unicode.
- Wrap generated visuals in semantics that carry the text they replaced.
- Register licences for assets the same way you would for dependencies.
- Budget the bytes out loud instead of discovering them at release time.

The result is deliberately modest: one emoji, one animation, one switch. But it makes the chat feel a little more alive without making the code less predictable. That trade is usually the right one.

If you liked the layout side of this, the write-up on [hiding chrome while scrolling](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-11-hide-status-bar-while-scrolling-flutter.html) covers the same principle from a different angle: motion should respond to what the reader is doing. And the [floating glass chrome](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-13-floating-glass-chrome-flutter.html) piece shows how the rest of the interface keeps its overlays out of the way.

## FAQ

**Does every emoji animate?** No. Only the 85 in the registry. An unregistered emoji keeps the static glyph, which is the correct default: no animation is better than a broken one.

**What happens to `🔥🔥`?** It stays static. The feature is an exact single-emoji match, not a per-character replacement. That keeps the bubble predictable and the number of animations bounded.

**Does this change the message or how it is stored?** No. It is a client-side presentation swap. The text is unchanged, and the animation is chosen at render time.

**Can I turn it off?** Yes, with the **Animate emoji-only messages** switch, or by setting the app-wide motion level to off. Both fall back to the original 48 px glyph.
