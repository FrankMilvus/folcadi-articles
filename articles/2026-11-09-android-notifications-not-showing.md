---
layout: post
title: "Android notifications vanished: one stripped icon, one silent catch"
description: "Push accepted, permission granted, channel at max importance — yet no card. How release resource shrinking silenced every Flutter Android notification."
tags: showdev, flutter, android, debugging
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-09-android-notifications-not-showing.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-09-android-notifications-not-showing.png
---

The server logged that it pushed to one device. FCM accepted the message. The notification permission was granted, the channel sat at maximum importance, and the phone was awake. No card appeared — not in the foreground, not in the background, not after a force-stop and relaunch.

The desktop build still showed notifications, which turned out to be the most misleading clue of the week: on Linux they never take this path at all.

This is the story of a complete, silent notification outage in a Flutter app, the one line in `adb logcat` that explained it, and the three separate problems that had to be fixed before a push actually became a card on screen.

## Four things that were all true

When a notification does not appear, the usual checklist is short: did the server send it, does the device have permission, is the channel enabled, is the app allowed to run? Every answer was yes.

- The backend reported a successful send to a single token.
- `POST_NOTIFICATIONS` was granted for the app.
- The Android channel was created at `Importance.max`.
- The device was online and the FCM token was current.

Any one of those failing would have been a satisfying explanation. None of them were failing. The gap was inside a step that every guide treats as a given: turning an accepted push into a rendered notification.

## The log line that ended the search

The client builds its own notification rather than letting the OS render the FCM payload, because it needs custom actions, per-account routing and quiet-hour rules. So the last step happens in Dart, through a local-notifications plugin.

A live `adb logcat` while sending a test push showed the app doing exactly what it should — and then failing:

```text
📱 [NotificationService] Showing notification id=… title=Folcadi test
⚠️ [NotificationService] show failed: PlatformException(invalid_large_icon,
   The resource ic_notification_large could not be found. Please make sure it
   has been added as a drawable resource to your Android head project.)
```

The first line is the app beginning to display a notification. The second is the plugin refusing to build it, because a drawable it was told to use did not exist in the installed APK.

This is the important detail: a missing **large icon** does not degrade a notification. The plugin throws while constructing it, before anything is posted. The app's `_show` helper catches and logs that exception — deliberately, so a failing notification can never take a message down with it — and the card never exists.

Every message alert and every call ring went through the same path. One missing file silenced all of them.

## The release build was not the same app

The source tree had `ic_notification_large` in every density folder. A debug build showed notifications. A release build did not.

That difference is the fingerprint of resource shrinking. Flutter's Gradle plugin enables it for release apps (`isShrinkResources = true`), and the shrinker keeps a resource only when it can see a reference to it. A `drawable` referenced from the manifest, from XML, or from generated `R` code is visible. A drawable referenced *only by name from Dart* is not:

```dart
largeIcon: DrawableResourceAndroidBitmap('ic_notification_large'),
```

To the shrinker, that string is data. It has no way to know it is also a resource identifier. So the file was removed from the release APK. Inspecting the shipped build made it obvious: `ic_stat_folcadi` was present — the manifest references it — and `ic_notification_large` was simply absent.

The same sweep removed the custom ringtone. It is referenced by name too, through `RawResourceAndroidNotificationSound('folcadi_ringtone')`, so the incoming-call channel pointed at a sound that no longer shipped. The call path was broken in a second, quieter way.

## The fix: stop naming a resource that must exist

There are two honest ways to keep a name-only resource in a release build: make it visible to the shrinker, or stop depending on the resource at all. The fix uses both, for different reasons.

The large icon now comes from the Flutter asset bundle, which cannot be stripped:

```dart
static ByteArrayAndroidBitmap? _cachedBrandLargeIcon;
static bool _brandLargeIconTried = false;

static Future<ByteArrayAndroidBitmap?> _brandLargeIcon() async {
  if (_brandLargeIconTried) return _cachedBrandLargeIcon;
  _brandLargeIconTried = true;
  try {
    final data = await rootBundle.load(
      'assets/brand/folcadi_brand_transparent.png',
    );
    if (data.lengthInBytes == 0) return null;
    _cachedBrandLargeIcon = ByteArrayAndroidBitmap(
      data.buffer.asUint8List(data.offsetInBytes, data.lengthInBytes),
    );
    return _cachedBrandLargeIcon;
  } catch (e) {
    debugPrint('⚠️ [NotificationService] brand large icon unavailable: $e');
    return null;
  }
}
```

Three properties matter here. Assets ship with the app, so the shrinker cannot remove them. The bytes are cached for the process, so a busy chat does not decode the same PNG for every message. And a load failure returns `null`, which means "no large icon" — not "no notification". A decorative image should never be able to cancel the alert it decorates.

The ringtone and the status-bar glyph still need to be real Android resources, so those are protected explicitly. Android's resource packaging has a documented escape hatch for exactly this case:

```xml
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@drawable/ic_stat_folcadi,@drawable/ic_notification_large,@raw/folcadi_ringtone" />
```

The comment above that line explains why the file exists, because the next person will otherwise delete it as redundant. It is not redundant; it is the only thing telling the shrinker that those names are load-bearing.

A regression test now reads the Dart sources, extracts every `DrawableResourceAndroidBitmap` and `RawResourceAndroidNotificationSound` name, and asserts each one appears in `keep.xml`. Add a new resource by name and forget the list, and the test fails before the release build can ship a silent outage.

## Two more reasons it stayed silent

Fixing the icon made the next test notification appear. It also exposed two problems that would have kept some users quiet even after that fix.

**The permission was never requested.** A one-time consent sheet had replaced the old startup permission request. The sheet only asked inside "Save", and dismissing it with "Not now" skipped the request entirely — while the device token stayed registered. Android 13+ then drops the notification without showing anything. The app now asks once per install, as soon as the first route is on screen, and remembers a denial so it does not nag. The rule is a small truth table:

```dart
static bool shouldRequestStartupPermission({
  required bool osEnabled,
  required bool alreadyAsked,
}) => !osEnabled && !alreadyAsked;
```

Notification settings also checks the OS permission when opened and when the app resumes, and shows a warning with a direct link to the system screen when notifications are off for the app. A setting the user cannot see is not a setting.

**The token table was full of tokens from the old app identity.** FCM registration tokens are scoped to a Firebase project. The project had changed as part of the rebrand, so tokens minted by the previous project could never be delivered to again. A dry-run validation of all stored tokens found 7 usable, 42 from the old project and 2 from uninstalled apps.

The runtime token pruner deliberately keeps "sender ID mismatch" rows, and that is the right default: a mismatched service account looks exactly like an old-project token, and auto-deleting on that signal could wipe the whole table if credentials were misconfigured. So cleanup is explicit and opt-in. A small maintenance script validates each token with a dry run, reports by default, and only deletes two error classes — old-project tokens and unregistered tokens — when told to apply. Every other failure is kept and reported.

Deleted tokens come back on their own: the device re-registers on its next login or restore, or when the notification toggle is flipped. Cleanup removes rows that could never work; it does not remove a working device.

## What "push accepted" actually proves

The most reusable lesson is narrower than "test your notifications".

**FCM accepting a message proves delivery to Google, not display to a user.** Everything after acceptance — permission, channel, payload shape, local rendering — is the app's job. A successful send log is the start of the investigation, not the end.

**A catch-and-log in a display path can hide a total outage.** Swallowing an exception is usually correct for a notification: one failed card must not break the message flow. But the same choice means the failure is invisible unless someone reads the log. The fix is not to throw; it is to make the display path observable and to test the thing that can silently break it.

**Release-only bugs need a release build.** The debug build was healthy. Resource shrinking, minification and asset packaging only diverge in release, so a notification test that passes in `flutter run` says nothing about the APK users install.

**A rebrand is a data migration too.** Renaming the app changed its Firebase project, and every stored token became project-scoped dead weight. The token table looked populated; most of it was archaeology.

## A short checklist for the next silent notification

When a push is accepted but no card appears, work backwards through the render path:

1. Confirm the client is building the notification itself, and read its logs on a real device.
2. Check the OS permission, not just the in-app setting.
3. Check the channel's importance and whether it was created before the user disabled it.
4. Reproduce on a release build, not only in debug.
5. If it fails only in release, suspect resource shrinking and inspect the APK's resource table.
6. Audit the token table if the app's identity or Firebase project has changed.

Steps 1 and 4 are the ones that would have saved the most time here.

## FAQ

### Why did desktop notifications keep working?

Because they are a different transport. On desktop the app receives a WebSocket event and asks the system notification service to show a card. On mobile the server sends an FCM push and the app renders a local notification. Desktop working was evidence that the server could reach the account — not evidence that the mobile display path was healthy.

### Is resource shrinking always the cause?

No, but it is a common cause of release-only failures. Any resource referenced only by name from Dart is a candidate: drawables, raw sounds, and anything else loaded through a string. If the same code works in debug and fails in release, compare the APK's resources before changing application logic.

### Why keep the large icon at all?

Because a notification has two icon slots. The small one is an alpha mask and can never show colour; the large one is where the actual brand artwork appears. Android treats a missing large icon as a fatal construction error in this plugin, so the choice was never "icon or no icon" — it was "a reliable icon or a silent outage".

### Will deleting old tokens break anyone?

Only devices that could not receive notifications anyway. A valid token re-registers on the next login or restore, and the app registers a fresh token when the notification toggle is turned on. The cleanup keeps every row it cannot classify with confidence.

### Does this affect iOS?

The shrinking issue is Android-specific because it concerns Android drawables and resource packaging. The permission problem is not: the same startup check reads the OS authorization status on iOS, and the settings warning appears there too. The token rules are shared.

## The pattern

Strip out the app-specific parts and the shape is small:

1. **A successful send is not a rendered card.** Follow the path all the way to the display call.
2. **Never let decoration cancel an alert.** A missing image should mean "no image", not "no notification".
3. **Name-only resources need an explicit keep list.** The shrinker is doing its job; tell it what the job cannot see.
4. **Request the OS permission outside any dismissible flow.** Once per install, with a way back from Settings.
5. **Treat a changed app identity as a data migration.** Tokens, caches and identifiers that were scoped to the old identity are now stale.

None of that is exotic. The reason it mattered is that each failure was individually reasonable, and together they produced a week of silence that looked like "push is broken" from every angle except the one place the notification was being built.
