---
layout: post
title: "The app that never started: a Flutter Wayland deadlock"
description: "Sometimes the app launched to nothing at all: no window, no splash, no log. The diagnosis came from two thread dumps — and the fix is XWayland, for now."
tags: showdev, flutter, linux, wayland
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-22-flutter-wayland-launch-freeze-x11-fallback.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-22-flutter-wayland-launch-freeze-x11-fallback.png
---

Sometimes the app did not start. Not a crash, not an error dialog — nothing. No window, no splash, no log line, no spinner. Clicking the icon again a few seconds later worked, which is exactly what makes this class of bug so easy to misread: it looks like a slow cold start, so you learn to click twice.

It is not a slow start. It is a deadlock, and the app was never going to appear no matter how long you waited.

## What a hang looks like from the outside

Three observations eventually lined up:

1. The launch produced no window at all. Not a white window, not a frozen splash — nothing was ever mapped.
2. The process was not idle. A launch that had hung sat at roughly ten percent of one core, for minutes.
3. A second launch worked. That is what let it survive in the wild: the workaround is one extra click, and the failure leaves no trace unless you go looking.

The third point is the trap. A bug whose workaround is "try again" gets reported as "sometimes slow", and slow starts do not get debugged.

## The log tells you nothing; the threads do

The app had no output, because a desktop launch sends stdout and stderr to nowhere by default. There was nothing to read. But the process was still alive, so there was a different source of truth: its threads.

Two full thread dumps, ten minutes apart, were identical. That is the definition of a hang — a live process that has stopped making progress.

The raster thread, the one that submits frames to the GPU:

```
ppoll
wl_display_dispatch_queue_timeout
wl_display_roundtrip_queue
???  (Mesa EGL, Wayland platform)
driBindContext
eglMakeCurrent
```

And the GTK main thread, the one that owns the window:

```
syscall  (futex)
g_mutex_lock
???  (Flutter's GTK embedder)
gtk_widget_size_allocate
```

Read together, they describe a cycle. The raster thread is inside a Wayland roundtrip during `eglMakeCurrent`, holding the renderer's frame lock. The main thread is inside size allocation, blocked on that same lock. Each is waiting for the other, and neither will let go.

That also explains something the symptom did not: the native splash could not save us. The window is mapped with that splash before Flutter paints anything — but mapping goes through the same GTK main loop, which is inside the deadlock. Even the runner's fifteen-second "remove the splash anyway" fallback is a timer on that loop. A frozen main loop freezes the safety net with it.

## The part that made it not our bug

The deadlock is in the Flutter engine's Linux embedder, and it arrived with a specific release. Flutter 3.50 made a new Wayland renderer the default: frames are presented to a child subsurface instead of being copied through GTK. It is a real improvement when it works — but on this release it also introduced two lock cycles, both tracked upstream:

- an intermittent freeze at startup, with exactly the stack above (raster thread spinning in `eglMakeCurrent` / `wl_display_roundtrip_queue`, main thread in `syscall`), reported across several compositors and GPU drivers ([flutter#196529](https://github.com/flutter/flutter/issues/196529));
- a deterministic deadlock on the **second** window resize, reported on niri ([flutter#196663](https://github.com/flutter/flutter/issues/196663)).

The same app on the previous stable release does neither. Once we matched our thread dump to the upstream report line for line, we stopped looking for a bug in our own code.

It is worth being precise about what that means. There was nothing to fix in our Dart, our plugins, or our startup sequence. Our app was the messenger: it launches a window, and the renderer that arrives with the window can wedge itself.

## Why we did not ship a retry

The obvious product answer is a watchdog: if no frame has been presented after N seconds, kill the process and start it again. It is also the wrong answer here, for four reasons.

**It is recovery, not a fix.** Every affected launch still costs the user the full timeout, then a restart. Making the bug survivable is not the same as removing it.

**The detector lives inside the corpse.** A Dart timer keeps running while the GTK and raster threads are deadlocked, so a watchdog would fire *for this particular cycle*. But it fires only because one specific pair of threads is stuck. A variant that also blocks the UI thread would leave the watchdog silently armed and useless, which is worse than no watchdog: it looks like protection.

**The relaunch races the zombie.** The dead process is still alive, still holding the app's data directory, preferences and session. Starting a second instance before the first is really gone is how you get two writers and a corrupted settings file. Making that safe means a reliable kill and hand-off, plus argv, working directory and environment handling for a bundle executable — a lot of machinery for a temporary upstream bug.

**It never leaves.** When upstream ships the fix, the watchdog has to be removed, or every launch keeps a self-restart mechanism nobody can connect to anything.

## What we shipped

An environment answer to an environment bug: **start the app on XWayland when the session offers it.**

On a Wayland session that also has an X display, the runner sets `GDK_BACKEND=x11` before GTK initializes. That selects the long-stable GTK renderer, which presents frames the way every pre-3.50 build did. The subsurface renderer is never instantiated, so neither the startup lock nor the resize lock exists. The rule is deliberately narrow:

- only when the session has both a Wayland display and an X display, so a Wayland-only desktop still starts normally;
- never over a backend the user chose: `GDK_BACKEND=wayland` keeps native Wayland, and is the opt-out;
- one log line when it happens, so the decision is visible instead of magic.

Native Wayland is the better renderer, and we want it back. This buys us a working app until the engine fix reaches our Flutter pin, and it is two small changes to delete afterwards.

There was a second, smaller surprise, and then a third.

On X11, GTK's template logic gives the window a traditional title bar instead of its own header bar whenever the window manager is not GNOME. That is fine where a window manager draws decorations — and useless on a Wayland compositor whose XWayland layer draws none. The first XWayland build came up with no title bar and no close button at all.

The obvious patch was to keep GTK's header bar on any XWayland window. It restored the close button and looked wrong: the reason the window had no frame in the first place is a decoration protocol that only Wayland clients negotiate, so the compositor's "no decorations" setting was simply ignored. GTK drew its own title bar and its client-side shadow ring, and with the theme falling back to a stock light Adwaita it read as a white frame around an app that is otherwise black.

The same session had the other half of the problem. GTK cannot scale fractionally, and on X11 the widget scale factor comes from one environment variable that defaults to 1. The compositor's 1.8 scale reaches X11 only as a DPI value, which GTK applies to font metrics rather than to widgets, so the engine reported a device pixel ratio of 1.0 and laid the UI out at the full physical width: a whole app, drawn about 45 % too small, inside a correctly sized window.

Both are handled in the runner now. When XWayland has no window manager behind it — a compositor's satellite, which is exactly the decoration-less case — the window stays borderless, so it looks the way it does on native Wayland. And the X11 DPI is rounded to the nearest integer and exported as `GDK_SCALE` before GTK starts, so the window scales like any other HiDPI X11 app. At 1.8 that means 2: one step wide, which is the closest an integer-scaled toolkit gets.

## How we checked it

The fix was verified on the machine it was diagnosed on — a Wayland session on niri with Mesa and an AMD iGPU, which is one of the reported environments:

- repeated launches, each one mapping its window (the failing behaviour was the *first* launch of a session);
- the automatic backend switch visible in the log, with no `GDK_BACKEND` set by hand;
- the window reaching the app's first screen;
- and, after the scale and decoration follow-up, a borderless window whose widgets match the output scale — the same screen captured at a pixel ratio of 1 and at 2 is the difference between a shrunken app and the intended one.

One honest limit: this bug is intermittent, so a few successful launches are evidence, not proof. What makes the claim stronger is that the deadlocking code path is not merely avoided by timing — the fallback selects a renderer that does not contain the cycle at all.

## What happens next

The workaround has an exit condition, written down with the code so it does not quietly become permanent:

1. Upgrade Flutter only to a release that actually contains the upstream fix, and read the notes rather than trusting the version number.
2. Rebuild and run with the fallback disabled, on Wayland.
3. Launch it many times — the startup failure is intermittent — and then resize the window twice, since the second resize is the deterministic case.
4. Only then delete the fallback, the scale derivation and the borderless rule.

Until then, the app starts the reliable way.

## The lesson that outlives the bug

A process that renders nothing, logs nothing and still burns CPU is not idle. It is doing something, and on Linux you can always ask it what: two thread dumps a few minutes apart will tell you whether it is working or waiting, and on what. In this case the answer — a renderer lock and a compositor roundtrip holding each other — was the entire diagnosis.

Two thread dumps beat one. And a workaround that names its own exit condition beats a clever one that does not.

If you are building a Flutter desktop app on Wayland: check which renderer your engine release selects before you blame your own code for a launch that never completes.
