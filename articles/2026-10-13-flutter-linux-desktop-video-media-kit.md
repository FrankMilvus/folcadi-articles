---
layout: post
title: Playing video on Flutter Linux desktop with media_kit
description: Flutter Linux desktop video had no backend, so every bubble wrongly said it expired. We fixed it with media_kit behind one abstraction.
tags: flutter, linux, desktop, showdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-flutter-linux-desktop-video-media-kit.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-13-flutter-linux-desktop-video-media-kit.png
---

Video is the one media type in **GwinG** that cannot fail quietly. A photo that
won't load falls back to an icon tile and most people never notice; a video that
won't play looks like the whole app is broken. So when every video bubble on our
Linux desktop build started showing **"Video expired"** — the label we reserve for
a file the server no longer has — we knew something was off. Flutter Linux desktop
video simply had no player behind it: `video_player` ships no Linux implementation,
and `media_kit` was not wired up yet. This is the story of finding that out, and
moving playback behind one abstraction so the chat widgets never have to know which
player is underneath.

It turned out to be less about adding a library and more about being honest with the
UI: the app wasn't failing to *find* the video, it was failing to *play* it, and the
error path told the user the wrong story.

## The symptom: every video bubble said "Video expired"

The report was suspiciously uniform. Not "some videos fail", not "big videos fail"
— **every** video, in every chat, on the Linux desktop build. That uniformity is
usually the clue you want: a codec problem, a corrupt file or a flaky network all
produce a mess of mixed results. An all-or-nothing failure means the problem is
above the file, in the layer that is supposed to decode it.

The other half of the clue was that the download was fine. Files downloaded
completely, landed in the cache, were playable by any other tool on the machine —
and the bubble still said *Video expired*. The label was a lie the app told itself.

## Why Flutter Linux desktop video had no backend at all

Here is the root cause, and it is almost embarrassingly simple.

The `video_player` plugin ships platform implementations for Android, iOS, macOS
and web — that is `video_player_android`, `video_player_avfoundation` and
`video_player_web`. There is **no** `video_player_linux`. On Linux, calling
`VideoPlayerController.initialize()` does not fail because the file is missing or
malformed; it throws `UnimplementedError`, because the method channel has nobody on
the other end.

Our video widget's control flow made that throw look like an expiry:

1. The auto-download step runs first. It is pure Dart (`http`), so it succeeds on
   Linux exactly as it does everywhere else. The file is on disk.
2. `initialize()` is then called on that local file and throws `UnimplementedError`.
3. The streaming fallback runs, and throws for the same reason.
4. The outer `catch` sets `_errored = true`, and the error branch renders our
   fallback tile — the one labelled **"Video expired"**.

Nothing expired. The file was there. The player did not exist. We had a UI that
described a server-side problem while the real problem was that we had never given
Linux a decoder.

## One interface, two players

We could have fixed this by special-casing Linux all over the widget. We didn't
want that, and we didn't want the reverse either — dragging `media_kit` into
Android, iOS, macOS and web where `video_player` already works perfectly. The two
chat widgets (`video_player.dart` and its fullscreen sibling) were written against
`VideoPlayerController`, and we wanted that to stay true.

So we put a seam between "the widget" and "the player": a single
`VideoPlaybackController` that both backends implement. It extends `ValueNotifier`
and mirrors the `video_player` API the call sites already used, which is what made
the migration mechanical rather than a rewrite:

```dart
/// A video surface and its transport, behind a single API.
///
/// Extends [ValueNotifier] so it is listened to and read exactly like a
/// `VideoPlayerController` — the call sites were written against that type.
abstract class VideoPlaybackController
    extends ValueNotifier<VideoPlaybackValue> {
  Future<void> initialize();

  Future<void> play();
  Future<void> pause();
  Future<void> seekTo(Duration position);
  Future<void> setPlaybackSpeed(double speed);

  /// The video surface to place in the widget tree.
  Widget buildView();
}
```

The pick between implementations is a one-line, platform-checked function, so
nothing downstream branches on the platform at all:

```dart
/// Whether video playback on this platform runs through `media_kit` (libmpv)
/// instead of the `video_player` plugin.
bool get videoUsesMediaKit => isLinuxDesktop;

VideoPlaybackController createVideoPlaybackController({
  Uri? networkUrl,
  String? filePath,
}) {
  return videoUsesMediaKit
      ? MediaKitVideoController.create(
          networkUrl: networkUrl,
          filePath: filePath,
        )
      : PluginVideoController.create(
          networkUrl: networkUrl,
          filePath: filePath,
        );
}
```

Two thin wrappers sit behind it. `PluginVideoController` is a pass-through to the
`video_player` plugin and is used on Android, iOS, macOS and web — unchanged.
`MediaKitVideoController` owns a media_kit `Player` plus a `VideoController`, and
maps `player.stream.*` (playing, position, duration, video params) onto our small
`VideoPlaybackValue` snapshot. It renders the surface with
`Video(controls: NoVideoControls)` because the app draws its own controls; letting
media_kit draw a second set would have produced two overlapping transports.

The widgets now call `createVideoPlaybackController(...)`, render
`controller.buildView()` instead of `VideoPlayer(...)`, and are otherwise identical
to before. `MediaKit.ensureInitialized()` runs once at startup, and only on the
platform that needs it.

## media_kit, without dragging every platform along

The obvious way to add media_kit is its aggregate package,
`media_kit_libs_video`, which bundles the native libraries for all supported
platforms. That is exactly wrong for us: it would pull native video libs into the
Android, iOS, macOS and web builds that already use `video_player` and are already
working. We chose the platform-scoped `media_kit_libs_linux` instead, so only Linux
gets the extra native payload.

One honest caveat we noted in the changelog: `media_kit_video` itself declares
native plugins for every platform, so it is still compiled into the non-Linux
builds even though only Linux uses it. That is a small size cost we accepted for
the convenience of a single pubspec dependency. If it ever matters, the clean fix
is to move the media_kit implementation behind a conditional import; for now the
cost is not worth the extra moving part.

If you're looking for the package, the [media_kit on pub.dev](https://pub.dev/packages/media_kit)
docs are unusually good, and Flutter's [desktop support page](https://docs.flutter.dev/platform-integration/desktop)
is worth a read before you assume a plugin works on every target.

## The packaging reality: libmpv and libepoxy

Adding a player is not just a Dart dependency — on Linux, `media_kit` renders
through the system **libmpv** and OpenGL via **libepoxy**. That makes them hard
runtime requirements of the shipped app, and there are two ways to ship on Linux:
a `.deb` and a portable tarball. Both had to learn about the new libraries.

The `.deb` declares its runtime dependencies in `Depends`, and a missing entry does
not fail the build — it makes the installed app die at startup with a cryptic
"error while loading shared libraries". So the dependency list now includes the mpv
and epoxy packages, derived the same way as every other entry: by reading what the
binary actually links against. The CI workflow installs the `-dev` packages to build
and then verifies the `mpv` and `epoxy` `pkg-config` modules are present, so a build
machine that can't satisfy them fails early instead of shipping a bundle that can't
start.

The portable install path can't rely on a package manager to resolve anything, so
when its post-install `ldd` check finds an unresolved library it prints a
distribution-specific hint naming the mpv/libepoxy packages per distro. That is the
same fail-soft posture the rest of our desktop support takes, and it's covered
alongside Google sign-in and session persistence in our desktop support notes if you
want the wider picture.

## What we could verify, and what we couldn't

We are careful to say what is proven and what is assumed. On the Linux side we
verified: the release build succeeds; `media_kit_video` and `media_kit_libs_linux`
show up in the generated plugin registrant; the binary links `libmpv` and
`libepoxy`; the bundle launches, prints the Linux libs as registered, restores a
session and connects; and `flutter analyze` is clean on the changed files.

What we could **not** verify in CI is the thing users actually care about — decoding
a real video on Linux, because that needs a real video message and a live session on
a real desktop. The build, the linking, the plugin registration and the startup path
are verified; the decode path is not, in automation. We say that plainly rather than
claiming a fix we haven't watched play.

The existing tests stayed green, which is a small reassurance that the abstraction
didn't change behaviour on the platforms it wasn't meant to touch: on the test
platform `videoUsesMediaKit` is false, so the fullscreen widget's close test still
exercises the stubbed `video_player` path — which throws, exactly as it did before.

## The limits we are keeping on purpose

Two honest gaps, because a post that only lists wins is not useful:

- **Windows still routes through `video_player`.** There is no Windows
  implementation either, so videos there are broken in the same way — and untested,
  because Windows has never been run on real hardware. The follow-up is to drop
  Windows into the same `media_kit` branch (plus its Windows libs) once we can
  actually test it. We deliberately didn't bolt it on blind.
- **Thumbnails and clip compression have no Linux backend.** The `video_thumbnail`
  and `v_video_compressor` plugins don't cover Linux, so first-frame extraction and
  pre-upload compression stay best-effort and fail soft: the bubble simply keeps its
  icon tile. That's the correct degradation, and it's unchanged by this work.

## FAQ

**Why did the video bubble say "Video expired" when the file was fine?**
Because the auto-download succeeded and the *player* was missing. `initialize()`
threw `UnimplementedError` on Linux, the widget's outer `catch` treated any
initialization failure as "unavailable", and its error branch is the tile labelled
"Video expired" — a label written for a server-side expiry, reused for a client-side
gap. The fix in the UI is that Linux now has a real player; the lesson is that error
copy should describe what actually failed.

**Why media_kit and not a `video_player` Linux package?**
There isn't one to install — `video_player` is implemented for Android, iOS, macOS
and web only, and the plugin's contract is that unsupported platforms throw. media_kit
is the pragmatic Linux backend because it renders through libmpv, which the distros
already package.

**Do the chat widgets know they're on Linux now?**
No, and that's the point. They call `createVideoPlaybackController(...)` and render
`controller.buildView()`. The only place that branches on the platform is
`videoUsesMediaKit`, and the only place that initializes the backend is a single
startup call.

**Does adding media_kit bloat the mobile builds?**
A little. We avoided the aggregate libs package so the native video libraries land
only on Linux, but `media_kit_video` still compiles its plugin scaffolding into the
other platforms. It's a small, accepted cost; a conditional import is the escape
hatch if it ever grows.

**Will Windows work now too?**
Not yet. Windows has no `video_player` implementation and has never been tested on
real hardware, so we left it on the broken-but-known path and listed moving it onto
media_kit as the follow-up rather than shipping an untested change.

If you want the shape of the desktop effort around this, we wrote up
[shared media and albums](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-shared-media-gallery-in-flutter.html)
and
[photo albums](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-photo-albums-in-flutter.html)
the same day — and the other half of running a desktop app is keeping people signed
in, which is its own story in
[making the Linux session survive a reboot](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-linux-keyring-session-durability.html).
If you're new here, start with
[what GwinG is](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html).

— the GwinG team
