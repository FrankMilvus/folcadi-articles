---
layout: post
title: Fast account switching in Flutter without a sign-in round trip
description: "How a Flutter messenger remembers accounts on-device, switches sessions in one tap, and keeps each account's messages, keys, drafts and tabs isolated."
tags: flutter, dart, showdev, languageexchange
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-17-fast-account-switching-in-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-17-fast-account-switching-in-flutter.png
---

Most messenger accounts are single-account by design: you sign in, the session
lives in one token slot, and every local cache is global. That works until a
real user needs two accounts on the same device — a personal profile and a
work one, or two language-learning identities.

In **GwinG**, a Flutter language-exchange messenger, we added **saved
accounts**: the app remembers up to five accounts and switches between them in
one tap, without a browser or a Google round trip. The interesting part was not
the picker. It was making "switch account" safe — the new session has to start
clean, and none of the previous account's data may leak into it.

This post covers the UX we landed on, the session lifecycle, and the isolation
patterns that made fast switching reliable.

## What the user sees

Saved accounts are listed on the login screen above the sign-in button, and in
an account switcher reachable from three places:

- **long-press** the profile avatar in the navbar (right-click on desktop);
- **`Alt+0`** (`Ctrl+0` if the user chose Ctrl as the main modifier);
- **More → Switch account**, next to Log out.

The switcher can switch to a saved account, add another one, remove a single
account, or sign out of everything. After a successful switch the app opens a
fresh home shell. That detail matters: the tab pages live in an `IndexedStack`,
so reusing the shell would keep the previous account's friends, chats and
search results mounted — and visible for a frame. Rebuilding the shell disposes
them instead.

Up to five accounts are remembered, most-recently-used first. Adding a sixth
evicts the least recently used account that is not the one being activated, and
removes its local data.

## Validate before you tear down

The tempting implementation is: read the target's token, drop the current
session, then validate. It is also the one that hurts users. A dropped Wi-Fi
connection at the wrong moment leaves them signed out of both accounts.

The rule we use is the reverse: **validate the target completely, then teardown,
then publish**.

```text
saved account tapped
        │
        ├─ access token still valid? ──no──► refresh the session
        │                                      │
        │                                      └─ rejected? → remove entry, show error
        ▼
  confirm the identity with the backend
        │
        ├─ network failure → keep the current session untouched
        └─ valid            → tear down → publish → reconnect
```

A network failure leaves the user exactly where they were. Only a definitive
rejection removes the saved entry and asks for a fresh sign-in.

The teardown itself is deliberately boring: end any active call, stop presence
polling, dispose the WebSocket, cancel background prefetchers, reset cached
encryption identity state, then clear in-memory tokens and notifiers. The
target's persisted data is **not** deleted by a switch.

## One session at a time

Two switches interleaving is a surprisingly easy bug to ship. We learned the
hard way that a "switch in progress" flag must be claimed **before the first
network await**, not after validation. Validation can take several round trips;
a second tap during that window otherwise passes the guard too.

We also stamp each session with an epoch. Late async work compares its captured
epoch with the current one and drops the result if the session moved on. That
covers the usual suspects — a profile fetch that finishes after a switch, a
token refresh that belongs to the previous account, a background cache refresh
that was already in flight.

There is a second half to the token-refresh case: the rotated token still
belongs to the account that requested it. We read the account id from the
refresh token itself and persist the result in that account's slot, never in
"whichever account is active now." A request that started under account A may
finish under account B, but it must not write into B.

## Scoping on-device data by account

This is where most of the work lives. A messenger accumulates a lot of local
state: messages, translations, pinned messages, chat tabs, drafts, encryption
keys, the upload outbox, friends. If any of it is keyed globally, the second
account sees the first account's data.

Every store now includes the account id in its key or path:

| Store | Scoped form |
|---|---|
| Message cache | one database per account |
| Translations / transcriptions | key suffix per account |
| Pinned messages | account + conversation |
| Conversation freshness markers | account + conversation |
| Chat tabs | account |
| Encryption keys | account prefix |
| Drafts and upload outbox | already per account |

The fallback segment is `_shared`, used only before a session exists. In normal
use there is always an account.

Encryption keys are the critical case. Sharing an identity between accounts
would present the same encryption identity to peers for two different users and
corrupt ratchet state on both sides. The cached identity is dropped on a switch,
and the next key operation reloads the target account's keys.

## Keeping async work on its account

Keying the stores by account is necessary but not sufficient. The subtle bug is
**when** the account is resolved. A cache write often happens after network
awaits:

```dart
final account = AccountScope.segment;
final fetched = await api.fetchMessages(conversationId);
if (AccountScope.segment != account) return; // the session moved on
await MessageCacheService.save(conversationId, fetched, segment: account);
```

Three details make this reliable:

1. The account is captured **once**, before the first await.
2. The result is dropped if the active account changed in the meantime.
3. The cache API accepts the captured segment instead of re-reading the global
   state at write time.

For encryption operations we go one step further and freeze the account for the
whole operation, across every nested await. Dart zones are a neat fit: the
operation runs in a child zone that carries the account, and key-store reads and
writes resolve that captured value instead of the live session.

```dart
static Future<T> capture<T>(Future<T> Function() body) {
  final captured = segment;
  return runZoned(body, zoneValues: {_operation: captured});
}

static String get segment =>
    (Zone.current[_operation] as String?) ?? _userId ?? '_shared';
```

This is the difference between "the key store is usually scoped" and "an
encrypt that started before a switch can never write into the next account."

## Migrating the single-account past

Existing installs had exactly one session: un-prefixed tokens, one message
database, one encryption identity, one tab layout. Updating must not log those
users out or orphan their encrypted history.

The migration runs only when the app has proven which user the legacy session
belongs to. It copies the tokens into that user's scoped slots, moves the
encryption keys and message cache into the same namespace, and then deletes the
legacy copies.

The important safety rule: a legacy layout is never claimed by "whichever
account loads first." If the old session cannot be restored at launch and the
user signs in manually, the app still migrates the legacy data to its real
owner and clears the old token slots — so a later launch can never silently
restore the wrong identity.

## Three bugs a review pass caught

The tests were green and the analyzer was clean, but a behavioural review still
found three isolation defects worth sharing:

- **A partial secure-store read looked like a complete session.** One token key
  was present and the other was not, so the read path skipped the fallback and
  returned a session with no refresh token. Fix: complete only the missing half
  from the fallback, and add a test for the mixed state.
- **Cache writes resolved the account at completion time.** A prefetch that
  started under account A could finish after a switch and write into account
  B's database. Fix: capture the account at the start and pass it through.
- **A legacy layout was claimed by the first account to load.** That exposed the
  previous user's tab names and chosen contacts. Fix: migrate legacy data only
  on the proven legacy path.

None of these are compiler errors. They are the class of bug that only shows up
when a real user switches accounts at an awkward moment.

## Lessons

- **Validate the destination before ending the current session.** Users forgive
  a slow switch; they do not forgive being signed out of both accounts.
- **Capture ambient state at the start of async work.** "Current account" is
  ambient state, and it can change under you.
- **Make the account an explicit parameter at the storage boundary.** It turns a
  silent cross-account write into a compile-time argument.
- **Treat legacy data as owned by a proven user.** Never let a random first load
  claim it.
- **Test the mixed and interrupted states**, not just the happy path: partial
  storage, a switch mid-fetch, a switch while a key operation is running.

## FAQ

**Does this work on desktop?**
Yes. Long-press or right-click the avatar, or press `Alt+0`. The same switcher
is available from the More tab.

**Do inactive accounts receive notifications?**
No. Push notifications follow the active account on that device. Switching back
restores its session and its pending sends.

**What happens to local data when I log out?**
A plain log-out removes the current account's tokens and its entry from the
picker, but keeps its namespaced local data so signing back in later restores the
encrypted history. Removing an account, or signing out of all accounts, deletes
that account's local data as well.

**How long does a saved account stay usable?**
As long as its refresh session is valid. If it expires, the picker asks you to
sign in again for that account instead of failing silently.

**Why only five accounts?**
It covers the realistic shared-device and personal/work cases while keeping the
picker and the local storage footprint small. The limit is a constant, not a
backend restriction.
