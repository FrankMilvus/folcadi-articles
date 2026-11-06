---
layout: post
title: "Your Flutter desktop app won't start: one corrupt preferences file"
description: "A truncated shared_preferences.json made a Flutter desktop app die before its first frame. The validation-and-quarantine fix, and the startup guard behind it."
tags: flutter, dart, desktop, debugging
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-07-corrupt-shared-preferences-flutter-desktop.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-07-corrupt-shared-preferences-flutter-desktop.png
---

The report was short: the desktop build opens and closes immediately. No window, no error dialog, nothing to click. On the terminal, the process printed a stack trace and quit before anything appeared on screen.

The cause was not a broken build, a missing library, or a failed update. It was one settings file with a missing bracket.

This post is about that failure and the fix: how a single corrupt JSON file can stop a Flutter desktop app before its first frame, why wrapping the read in `try/catch` does not save you, and how the app now repairs itself at launch without throwing away the evidence.

## A launch that dies before it can report anything

The terminal showed this:

```
Unhandled Exception: FormatException: Unexpected character (at character 31943)
...notif_suppressed_136":1793799452696}"flutter.keystore_durability_expecte...
```

Look closely at the quoted fragment. There is a closing `}` after a timestamp, and then another key starts immediately: `…2696}"flutter…`. That is not a value the app wrote. It is two JSON objects glued together — the remains of an interrupted write.

The exception happened inside an awaited call in `main()`, before `runApp()`. An async `main` without a top-level catch lets that exception escape, and the Flutter engine never gets a route to render. From the user's side, the app silently refuses to start.

There is no error boundary for this class of bug. The crash reporting tool had not finished starting either, so there was no report to read. The only evidence was the terminal output and the file on disk.

## Why one file decides whether the app starts

On mobile, `shared_preferences` is backed by platform-native storage. On Linux and Windows, the plugins are different: both keep the entire preferences store in a single file called `shared_preferences.json` in the app's support directory. On Linux that is:

```
~/.local/share/com.folcadi.app/shared_preferences.json
```

Every key the app saves lives in that one JSON object. The plugin does not read keys lazily; it decodes the whole file once, inside `SharedPreferences.getInstance()`, and hands you a map.

That design is fine for settings. It becomes a critical path when the first thing your app does is read a setting. Folcadi restores the user's appearance preferences before `runApp()` so the first frame is already in the right theme. That means launch order is:

1. Flutter bindings initialize.
2. The app asks for preferences.
3. The plugin reads and decodes the JSON file.
4. Only then does the app build its first widget.

If step 3 throws, steps 4 onward never happen.

This is also why the bug was invisible in development. A fresh profile has a valid file. Tests that mock preferences never see the real decoder. The failure needs a file that was damaged outside the app's control — a power loss, a forced shutdown, a full disk during a write, or a sync tool touching the profile directory. We cannot point to the exact event that damaged this one, and that is the point: the app must survive a damaged file regardless of how it happened.

## Why `try/catch` around the read is too late

The obvious fix is to catch the exception where preferences are read:

```dart
try {
  final prefs = await SharedPreferences.getInstance();
  // ...
} catch (e) {
  // use defaults
}
```

That helps the call site you wrapped, and it is worth doing — the app now has exactly that guard around its startup restores. But it does not repair the store. Every other call to `getInstance()` in the app still hits the same file and throws the same error. A settings screen opened ten minutes later fails in a different place, with a different stack trace, and the app looks broken in a new way.

Worse, the failed read leaves the file untouched. The next launch repeats the failure. A one-time catch turns a hard crash into a permanent, scattered malfunction.

The real problem is that the file is untrusted input. Treat it that way: inspect it before the plugin ever opens it, and remove it from the plugin's path when it is unusable.

## Validate before you initialize

The repair runs immediately after `WidgetsFlutterBinding.ensureInitialized()` and before any code touches `SharedPreferences`:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await repairCorruptSharedPreferences();

  // ...the rest of startup
}
```

The function has a simple contract. It returns `true` if a corrupt file was moved aside, and `false` if there was nothing to do — on web, on mobile, when the file is missing, or when the file is healthy.

Inside, the checks are deliberately narrow:

```dart
if (!await target.exists()) return false;

final raw = await target.readAsString();
if (raw.trim().isEmpty) return false;

try {
  final decoded = jsonDecode(raw);
  // The plugin expects a JSON object; anything else is not a usable store.
  if (decoded is Map) return false;
} on FormatException catch (e) {
  debugPrint('PrefsRepair: corrupt preferences file: $e');
}
```

Three details matter here.

**Whitespace-only is not corruption.** An empty file is what a fresh install or a cleared profile looks like. Renaming it would create noise and a pointless backup on every clean launch.

**Valid JSON is not automatically valid preferences.** A file containing `[1, 2, 3]` decodes without error but cannot be a preferences map. A type check catches that case, which a `jsonDecode`-only guard would miss.

**The check is cheap.** Reading one small file and decoding it once adds a few milliseconds to launch. It is negligible next to the network calls that used to run before the first frame, and it buys a guaranteed start.

Platform detection lives in the same function. It returns early unless the target platform is Linux or Windows, so mobile and web keep their native storage untouched:

```dart
if (kIsWeb) return false;
if (defaultTargetPlatform != TargetPlatform.linux &&
    defaultTargetPlatform != TargetPlatform.windows) {
  return false;
}
```

The file is injectable for tests, so the real plugin and the real app directory are not needed to prove the behavior.

## Quarantine instead of delete

A corrupt store is unusable. Deleting it would let the app start, but it would also destroy the only artifact that explains what happened. So the repair renames the file:

```dart
final backup = File(
  '${target.path}.corrupt-${DateTime.now().millisecondsSinceEpoch}',
);

try {
  await target.rename(backup.path);
  return true;
} catch (e) {
  // Rename can fail across filesystems; startup must still win.
  try {
    await target.delete();
    return true;
  } catch (deleteError) {
    return false;
  }
}
```

The timestamped suffix means repeated failures do not overwrite each other. The original bytes stay on disk for inspection, and the plugin creates a fresh empty map on the next read.

The delete fallback is not defensive noise. Renames can fail for reasons unrelated to the file itself — a permission change, a directory that is on another mount, a locked path. The priority order is explicit: **start the app first, preserve evidence second**. A failed repair returns `false`, which is honest, but the startup guard below means even that does not keep the app down.

## The second half: startup must degrade, not block

Repairing the file handles the known failure. The broader rule is that startup initialization is best-effort: a plugin failing should cost you a setting, not the whole app.

Four independent pieces of startup work run together in a `Future.wait`:

- window chrome (desktop only, no-op elsewhere),
- the saved appearance preferences,
- crash reporting / Firebase initialization,
- local notification setup.

All four are wrapped in a single `try/catch`:

```dart
try {
  await Future.wait<void>([
    WindowChrome.ensureInitialized(),
    restoreAppearancePrefs(),
    _initializeFirebase(),
    _initializeNotifications(),
  ]);
} catch (error, stack) {
  // A prefs/plugin failure must not prevent the app from starting:
  // the notifiers already hold their defaults.
  debugPrint('⚠️ Startup restore failed: $error\n$stack');
}
```

This is a deliberate trade-off. `Future.wait` fails fast: the first error propagates and the remaining futures keep running unobserved. The app starts with whatever defaults the notifiers already hold. A user might get the light theme instead of the dark one they chose, or miss one restored preference. They still get a working app, and the log names the failure.

The guard is not a substitute for the repair. If the file stays corrupt, every later preferences access still throws. The two layers solve different problems: one removes the bad input, the other guarantees the app does not hang on the splash screen while initialization fails.

## Four tests that matter

The repair has four tests, all against real files in a temporary directory:

| Case | Input | Expected |
|---|---|---|
| Malformed JSON | Two JSON objects concatenated | Renamed to `.corrupt-*`, contents preserved |
| Valid JSON, wrong shape | `[1, 2, 3]` | Quarantined |
| Healthy store | `{"flutter.theme_mode":"dark"}` | Untouched, no backup |
| Missing or blank | No file / whitespace only | Not treated as corruption |

The first test uses the actual shape from the crash log. Not an abstract invalid string — the kind of damage the app really saw. That matters, because the test documents the supported failure, not just the code path.

The healthy-file case is just as important. A repair routine that quarantines valid preferences would reset user settings on every launch, and it would look like a much worse bug than the one it fixed.

## What it costs the user

Honesty about the trade-off matters more than a clean story.

When the file is quarantined, **settings reset to defaults**: theme, notification preferences, keyboard options, quick-reaction emoji, and similar local state. The old values remain in the `.corrupt-<epoch>` backup, but the app does not attempt to salvage them. A partially decoded store is exactly the kind of guess that produces weird states.

The secure session store is untouched, so the keyring-backed login survives. But on desktops where the keyring never proved durable, a plaintext fallback token is kept in this same preferences file. That copy is lost with the file, so a user on such a machine may need to sign in again.

That is the trade: preferences are disposable, launch is not. A few lost settings beat an app that cannot open.

## If it happens to you

If a Flutter desktop app dies at launch with a `FormatException` from `shared_preferences`, you can recover it manually without waiting for a patch:

1. Close the app.
2. Go to the app's support directory. On Linux, `~/.local/share/<app-id>/`; on Windows, `%APPDATA%\<app-id>\`.
3. Rename `shared_preferences.json` to something else, or delete it.
4. Start the app again. It will come up with default settings.

Keep the renamed file if you want to diagnose what wrote the bad data. If the app starts and then fails somewhere else, the preferences file was not the cause — but a corrupt JSON file is one of the few startup failures with such a clean recovery.

## What I took away

The bug was not in the preferences code. It was in the assumption that a file written by the app is always readable by the app.

Three rules came out of it:

- **Any file the app reads at startup is untrusted input.** Validate the shape, not just the presence.
- **Repair at the boundary, not at every call site.** One guard before the plugin initializes is simpler and more complete than a catch in every screen.
- **Preserve the failure.** Rename instead of delete; a corrupt file is a bug report you cannot reproduce later.

The app still has plenty of startup work that can fail. The difference is that a failure now produces a running app with default settings and a log line, instead of a process that exits before the window appears.

## FAQ

### Does this affect mobile?

No. The repair returns early unless the platform is Linux or Windows. Android, iOS, macOS, and web use different storage backends and keep their existing behavior.

### Why not switch to a different storage package?

The file format was not the problem; the missing validation was. Whatever storage package you pick, a file on disk can be truncated, replaced, or synced by something else. The fix belongs at the boundary, so it applies regardless of the backend.

### Why keep the corrupt file instead of deleting it?

Because it is the only evidence. The backup is timestamped and costs a few kilobytes. If the corruption repeats, having the previous file is how you tell a one-off crash from a write pattern that keeps damaging the store.

### Can the settings be recovered from the backup?

Not automatically. The app deliberately starts from defaults rather than trying to decode a damaged map. For a one-off failure, most users will not care. For a recurring one, the backup is diagnostic material, not a restore point.

---

If you build Flutter desktop apps, the cheap version of this lesson is: check the files your app reads before the first frame. A missing bracket should never be enough to keep a window closed.
