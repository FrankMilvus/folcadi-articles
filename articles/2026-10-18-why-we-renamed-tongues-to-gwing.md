---
layout: post
title: Why we renamed tonGues to GwinG — and what stayed the same
description: Why tonGues became GwinG — a visible rebrand that keeps accounts, package identity, and encrypted chats exactly where they were.
tags: showdev, languageexchange, branding, opensource
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-18-why-we-renamed-tongues-to-gwing.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/why-we-renamed-tongues-to-gwing.png
---

## Why this post exists

**GwinG** is the new name of **tonGues** — the same language-exchange messenger, with the same accounts, conversations and encryption. The change you will notice is on the surface: the name, the wordmark, the tagline, the splash screen and the desktop builds. The change you should *not* notice is underneath: your sign-in, your history, your keys and the links you have already shared stay where they are.

If you are new to the project, the [overview of GwinG](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html) is the right place to start. This post is about the rename itself: why the old name had to go, how we chose a new one, and the line we drew between the visible brand and the things that must never move.

## The old name was already taken

The short reason for the rename is that there was already a language app using the same name. Not a similar name, not a name in the same category — the same word.

That is not a scandal, and it is not unusual for a small project. It is a practical dead end. A name is how people find you in a store search, how they recommend you in a sentence, and how they tell a friend which app to install. When two products answer to the same word, every one of those moments turns into a clarification: the right app, by the right developer, not the other one. For a project built around conversation, spending the first minute of every conversation explaining your name is a bad trade.

There was a second problem, smaller but persistent. The old tagline was a pun — “set tonGues wagging” — and a pun only works when the name is already familiar. It also depended on a spelling that is easy to fumble in a search box. Once the name had to stand on its own, the wordplay was carrying more weight than the word.

## What a name has to do when it is also a handle

For an app, a name has four jobs:

- **Be found.** Someone types it into a store search and sees the right result.
- **Be said.** Someone recommends it out loud without spelling it.
- **Be typed.** Someone enters it into a message without autocorrect fighting back.
- **Be owned.** A search for it points at this project, not at a dictionary entry or a different product.

The old name was decent at being said and typed. It failed the first and fourth jobs, and those are the ones that compound. Every new user who cannot find you is a user you never hear from.

## Choosing a name we could own

We wanted a coined name rather than a borrowed one. **GwinG** is not a dictionary word. It is five letters with a capital at each end, and that shape is the whole design: two bookends around a short, plain middle.

A coined name does something a common noun cannot — it makes the search result unambiguous. It also gives the wordmark a natural structure. The two `G`s are the edges, so they can carry the strongest colour, while the middle can recede.

We kept the name short enough to sit in an app bar without shrinking, and distinct enough to be a handle. That last point mattered more than elegance. A name you have to qualify is a name that never quite becomes yours.

## The wordmark: red at the edges, amber at the centre

The wordmark was rebuilt around that shape. Both `G`s are accented, and every letter is coloured by its distance to the nearest accent. The two `G`s sit at the ends, so they take the red end of the palette. The `win` between them blends through orange and lands on amber at the centre.

That is the rule in one sentence: **red at the accented edges, amber at the centre**. It reads as one word rather than two initials, and it uses the palette the app already had — the same red and amber, not a new set of colours bolted on for the occasion.

The change is deliberately local to the wordmark widget. Strings with a single capital still render the way they did before, so the same component keeps working on screens that are not the login page. In practice you see it on the login screen, the friends header, the More tab and the in-app splash — the places where the app says its own name.

## The tagline: where languages meet

The old tagline had to go with the old name, because the joke lived inside the word. The new one is **where languages meet**.

It is shorter, it does not depend on a pun, and it describes the product without naming a language or a direction. Two people can meet in the middle; neither has to switch to the other's language to start. That is the idea the app is built around, and the tagline now says it in four words.

It is localized alongside the rest of the interface — German, Spanish, French, Korean and Russian, with English as the source — so the line is not an English-only slogan pasted onto a translated app.

## A new splash with the same flame

The splash image was redrawn with the new wordmark, keeping the flame, the palette and the bold monospace styling. The native splash assets were regenerated for Android, iOS and web, so the first frame of the app matches the in-app splash instead of showing the old name for a beat.

The launcher icon is intentionally unchanged. It is a textless flame, and it never depended on the name. A new icon is a separate design decision, not something to smuggle into a rename at the last minute. The tool that generates the splash was also updated, so regenerating the assets reproduces the new wordmark instead of quietly restoring the old one.

## Desktop builds get a new name

The desktop builds were renamed from `tongues` to `gwing`. The binary, the packaged artifacts, the window title and the installer display name now say GwinG. In day-to-day use that means a `gwing` binary, `gwing` packages, and a GwinG entry in the launcher instead of the old one.

Upgrades were the part that needed care. The installers clean up the old executable and shortcuts, and the Linux package declares that it replaces the old one instead of sitting beside it. The previous `tongues-*` downloads remain available for one release cycle, so nobody is stranded mid-upgrade.

The important detail is invisible: the desktop storage identity was left exactly as it was. The executable changed; the data it opens did not. That is what keeps preferences, the secure store and the encryption identity intact across an upgrade.

## What did not change

This is the list we wrote down before touching anything:

- **Accounts and sessions.** Your account is the same account. There is no sign-out, no re-link and no migration step. If you use saved accounts, they are still saved — the [account-switching write-up](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-17-fast-account-switching-in-flutter.html) explains how that isolation works.
- **Package identity.** The installed app and the store listing are the same product. Updates arrive as updates, not as a second app with a new identity.
- **Encrypted chats.** Encryption was not touched. The encryption identity and keys stay the same, so existing encrypted history remains readable.
- **Deep links.** The `tongue://` scheme is unchanged, so links already shared in chats, notes or bookmarks keep resolving.
- **Notification channels.** The identifiers are unchanged; only the display names now say GwinG. Per-channel sound, vibration and mute settings survive.
- **Local data.** Theme choice, saved accounts, cached media and local history keep their storage keys, so an upgrade does not orphan them.
- **Article URLs.** Existing posts keep their slugs and canonical addresses. Bookmarks and inbound links do not break.
- **Historical records.** Old changelogs and plans still say tonGues. They are records of what was true then, and rewriting them would make the history less accurate, not more.

The pattern is simple: **the name is allowed to change; the identity is not.** A rebrand that quietly changes identity is not a rebrand. It is a new app wearing an old name, and users pay for it with a sign-in screen and a missing history.

## FAQ

**Is GwinG a different app from tonGues?**
No. It is the same app under a new visible name. Your account, friends, conversations and media are the ones you already had.

**Will I have to sign in again or lose my encrypted history?**
No. The rename does not touch accounts, credentials or the encryption identity. On desktop, the storage identity was kept specifically so an upgrade finds the same preferences, secure store and keys.

**Do old links and notification settings still work?**
Yes. The deep-link scheme is unchanged, so previously shared links keep resolving. Notification channel identifiers are unchanged too, so per-channel settings survive; only the display names now say GwinG.

## The test we cared about

A rename is easy to make loud and hard to make safe. The visible parts — the wordmark, the tagline, the splash, the desktop artifacts — are the parts people will notice. The invisible parts are the ones that decide whether the change feels like an update or like a reset.

Our test was simple: if you had a saved account, a shared link, a muted channel and an encrypted conversation from before the rename, all four should still be there afterwards. The name changed. Everything that makes the app yours did not.

— the GwinG team
