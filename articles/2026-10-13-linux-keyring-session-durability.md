---
layout: post
title: Staying signed in on Linux when the keyring doesn't survive a reboot
description: "Flutter Linux secure storage that survives a reboot: how we prove the OS keyring is durable before dropping the plaintext session fallback."
tags: flutter, linux, security, showdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-linux-keyring-session-durability.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-13-linux-keyring-session-durability.png
---

We keep people signed in on desktop the same way we do on mobile: the session is
an access/refresh token pair, and we would much rather store it in the operating
system's secure store than in a plaintext file. On Linux that secure store is the
keyring — libsecret talking to the Secret Service that `gnome-keyring`
implements. Getting **Flutter Linux secure storage** to behave on every machine
turned out to be harder than the plugin's tidy `write`/`read` API suggests. The
last bug we fixed was the most interesting one: a keyring that kept its promise
for exactly as long as the machine stayed switched on.

This is the story of that bug, why the "obvious" proofs that a keyring works were
all wrong, and the small piece of Linux kernel state that finally made the check
honest.

## The setup: Flutter Linux secure storage and a proof-gated fallback

On desktop, GwinG persists the session through `flutter_secure_storage`. On
Android and iOS that is the OS keystore, which is always present. On desktop it
is the OS *keyring* — libsecret / `gnome-keyring` on Linux, Credential Manager on
Windows — and a keyring is a much less certain thing. It can be missing, locked,
or reachable only from inside a graphical session.

The plugin makes this worse in a quiet way:
[`flutter_secure_storage`](https://pub.dev/packages/flutter_secure_storage) can
accept a write and keep nothing. No exception, no error — the call returns
normally and the value is simply not there later. A store backed by a transient
collection returns the value immediately and drops it when the process exits.

So we do two things:

- We write the tokens to the secure store.
- **Until that store proves it is durable, we also keep a plaintext copy** in
  `SharedPreferences`, so the session is always restorable.

The moment the store is proved durable, we delete the plaintext copy, and the
keyring becomes the only place the session lives. That "proved durable" step is
the whole game, and it is where both of this year's desktop session bugs came
from.

## The fallback is proof-gated, not permanent

A plaintext fallback that never goes away would be the simplest, safest thing for
the session — and we rejected it. It would leave a refresh token unencrypted on
every machine, including the many whose keyring works perfectly. That is the
wrong default.

Instead the fallback is **proof-gated**. We trust the secure store only after we
have evidence that it actually keeps values across launches, and until then we
keep the encrypted copy alongside the plaintext one. The evidence is deliberately
*not* a same-process write→read round-trip, which as we said can succeed against
a store that never persists. The proof is a value written on one launch and read
back on the **next** one:

```dart
// A value written now and compared on the next launch is the only proof that
// outlives the process that wrote it. A same-process round-trip proves nothing:
// a transient collection passes it and loses the value anyway.
final proofMatches = expected != null && seen == expected;
```

If the value we wrote last launch is still there this launch, the store kept
something across a restart, and we can trust it. That was the design after our
first desktop bug, and it fixed the "login screen on every launch" report. Then
came Linux.

## The bug: a keyring that only lives for the running session

A user on **niri/Arch** had `gnome-keyring` installed and running — it owns
`org.freedesktop.secrets`, via
[D-Bus activation](https://specifications.freedesktop.org/secret-service-spec/latest/)
— but its default collection was never unlocked at login, because niri starts the
daemon outside any PAM unlock.

The daemon accepted a write and returned it on the next read **within the same
session**. So our one-launch proof matched. The store looked durable. We deleted
the plaintext fallback.

But the value only ever lived inside the running daemon. After a reboot it was
gone — and so was the session, because we had already thrown away the only copy
that would have survived. The fallback had protected the user right up until the
keyring "passed the test", and then stopped.

The diagnostics on that machine told the story plainly: the app's keyring entry
held an **empty secret**, the real tokens sat in the plaintext file with the
marker recording that the last durable save went to `prefs`, and the proof value
in prefs matched the proof value in the keyring. Everything agreed that the store
was fine. The machine simply disagreed at the next boot.

## Why the one-launch proof was wrong

Our proof answered the question "did this value survive the process that wrote
it?" That is the right question for a store that can drop data when an app exits.
It is the wrong question on Linux, where a keyring can drop data when the
*session* exits — a different, later, more destructive event.

An app restart and a reboot are not the same boundary. On a machine whose keyring
caches writes in memory, an app restart inside one boot proves nothing about the
reboot that follows it.

## The fix: make the proof cross a boot

The fix is small and, in hindsight, obvious: the proof record now carries the
kernel **boot id**.

Linux exposes a fresh, random identifier for the running boot at
`/proc/sys/kernel/random/boot_id`. It changes on every boot. So we write the proof
as `"<bootId>:<nonce>"` — the boot the value was written in, plus a nonce that
makes it unique — and we trust the store only when a proof written in one boot is
read back **in a later boot**:

```dart
final bootId = await _linuxBootId();          // null off Linux, or unreadable
final proofMatches = expected != null && seen == expected;

// `expected` is "<bootId>:<nonce>". A record with no separator is a legacy
// value, which can never prove a reboot was survived.
final expectedBootId = _proofBootId(expected);
final crossedBoot =
    bootId != null && expectedBootId != null && expectedBootId != bootId;

_keystoreTrusted = proofMatches && (!isLinuxDesktop || crossedBoot);
```

A keyring that merely caches for the running session can never satisfy
`crossedBoot`: the proof it holds was written in the current boot, so the store
never reaches the durable state. The plaintext fallback stays, and the session
survives the reboot. A keyring that genuinely persists reaches durable exactly
one reboot after sign-in, then becomes the sole copy, as designed.

Two details matter here:

- **Legacy records can never pass.** A proof written by the old build has no `:`
  separator, so `_proofBootId` returns null and `crossedBoot` is false. An
  upgrade cannot falsely trust a store; the next launch simply replaces the
  record with the boot-id format.
- **The check is scoped to Linux.** Off Linux the platform keystore (Android/iOS
  Keystore, Windows Credential Manager) is reliable, so the boot-id requirement
  is disabled and a matching proof is enough — unchanged behaviour. We added
  `isLinuxDesktop` to `platform_support.dart` precisely to keep this to the one
  flaky platform, with `isDesktop` still covering Windows and macOS.

## Failing safe: an unreadable boot id keeps you signed in

`_linuxBootId()` returns null if it cannot read the file, and a null boot id can
never satisfy `crossedBoot`. That means a read failure is treated as "not
durable" — the conservative direction, the one that keeps the user signed in
rather than out. We would rather leave a fallback in place than risk trusting a
store we could not assess.

The same principle runs through the whole check: every uncertain outcome resolves
to keeping the session, and only positive, cross-boot evidence moves the app to
keystore-only.

## Why the obvious alternatives don't work

We considered the two shortcuts and rejected both:

- **Never drop the plaintext fallback on desktop.** Simplest, and it would have
  avoided this bug — but it leaves the refresh token unencrypted on every healthy
  machine, forever. We rejected it once before and kept it rejected; it trades a
  real, permanent weakening of secret storage for a bug that has a precise fix.
- **Require N consecutive launches.** More launches do not help. A daemon that
  keeps the value in memory passes any number of same-boot restarts; it only
  fails at the boundary a reboot represents. N consecutive launches only ever
  prove same-boot survival.

The boot boundary is the honest version of "durable across a restart": it is
precise, it keeps the keystore-only path for healthy machines, and it costs those
machines exactly one extra launch carrying a plaintext copy.

## What actually changes for users

Not much, which is the point:

- On a machine whose keyring never proves durable, the fallback stays and **the
  session survives reboots**. The user stays signed in, at the cost of an
  unencrypted local copy — which they can remove by unlocking the keyring's
  default collection at login.
- On a healthy keyring, the plaintext copy is kept for one launch cycle after
  sign-in, then dropped; after that the keyring is the only copy, as before.

The change is covered by a test that launches the app twice inside one boot (the
fallback is kept) and once in a later boot (the store is trusted and the fallback
dropped). The rest of the storage suite still passes, so the Linux path stayed
additive.

If you want the surrounding work, we shipped a
[photo album flow](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-photo-albums-in-flutter.html),
a [shared media gallery](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-shared-media-gallery-in-flutter.html)
and the [Linux desktop video and media kit](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-flutter-linux-desktop-video-media-kit.html)
the same day, and there's a
[full overview of GwinG](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html)
if you're new here.

## FAQ

**Does this log me out on a healthy Linux keyring?**
No. It keeps the plaintext copy for one launch cycle after you sign in, then drops
it once a proof crosses a reboot. After that the keyring is the only copy, exactly
as before.

**Why not just always keep the plaintext fallback on Linux?**
Because it would leave the refresh token unencrypted on every Linux machine,
including the ones whose keyring is perfectly fine. Staying signed in matters, but
not at the permanent cost of plaintext secrets everywhere.

**What if the app can't read the boot id?**
It treats the store as not durable and keeps the fallback. That is the safe
direction — we would rather keep you signed in than trust a store we could not
assess.

**Does this affect Android, iOS, Windows or macOS?**
No. Off Linux the boot-id requirement is disabled and a proof that survives a
restart is enough, because the platform keystore there is reliable. Legacy proof
records behave the same way.

**Is the boot id sensitive?**
No. It is a random per-boot identifier whose only use here is answering "is this a
different boot?" It is not tied to a user or a machine, and it carries no data
beyond the boot boundary itself.

— the GwinG team
