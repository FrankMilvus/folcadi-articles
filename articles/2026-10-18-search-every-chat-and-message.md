---
layout: post
title: Search every chat and every message from the chats list
description: GwinG's chats list now searches chat names instantly on-device and old messages on the server, then opens the hit scrolled into place.
tags: showdev, flutter, postgresql, webdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-18-search-every-chat-and-message.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/search-every-chat-and-message.png
---

## Two questions, one search box

A search box in a messenger is usually asked two different things. The first is **"which chat is this?"** — you half-remember a name, a nickname, or the last thing that was said. The second is harder: **"where did we talk about this?"** — you remember the topic, not the conversation, and the line you want is months old.

Most search boxes answer one and fake the other. In **GwinG**, we wanted both from the same place: the chats list. The list now has a search button at the top. Tap it and a query box slides in above the chats, replacing the brand header. The menu collapses, the floating action buttons hide, and the whole page becomes one field. The back arrow, `Esc`, or the system back button puts it away again.

That sounds like a small UI change. The work was in making the two questions share one result list without lying about either. If you are new to GwinG, the [overview post](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html) covers the app; this piece is about the search path, and it follows the chat-surface work in the [emoji, pins and grouped-bubbles write-up](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-16-emoji-shortcodes-pins-and-grouped-bubbles.html).

## Instant filtering, then a slower second opinion

The "which chat" half is answered entirely on the device, on every keystroke. The list rows already carry what we need: a friend's name, surname, nickname and display name; a group's name; the last message preview; the draft you never sent. Pending requests match on identity fields too. None of this needs a round trip, so the list narrows as you type.

The "where did we talk about this" half cannot work that way. The message you want may be far outside the handful of rows the list has loaded, so the app also asks a server-side message search endpoint. That call is **debounced by 300 ms**, so a fast typist does not fire one request per letter. A sequence guard drops stale responses: if an older query resolves after a newer one, its results are ignored instead of briefly overwriting the correct list.

The two layers merge into one view. While the server is still thinking, local matches stay on screen and a spinner sits where remote results will land. When there is nothing at all, the page shows a single "No chats found" state rather than two competing empty messages.

## What the server actually searches

The message search is deliberately narrow, because "search everything" is the wrong default when part of the history is encrypted.

- It only looks at conversations the caller belongs to.
- It skips conversations marked encrypted, and skips deleted messages.
- It matches the query as a case-insensitive substring of message content.
- It returns newest first, with a default cap of 50 hits.

Each hit arrives with enough context to render a row without another request: the conversation it belongs to, its title, the sender's identity, and the timestamp. A group sender is included for a practical reason — the client cannot resolve a member's display name from a message payload alone, so the server resolves it while it already has the row.

The hit payload reuses the same serializer as an ordinary message. That is a quiet but important decision: a search result carries the same reaction, reply and link-preview fields as the message does in the conversation, so the row can show what the message actually is instead of a stripped-down copy that drifts from the real thing.

## Messages above chats

Results are grouped into two sections, and the order is intentional: **Messages** first, then **Chats**.

A message row shows the chat title, `sender: snippet`, a group badge when relevant, and the date. It is a direct answer to "where did we talk about this", so it gets the top of the page.

The Chats section is the union of two matches. A row stays when its name, nickname, last-message preview or draft matches **or** when any server hit belongs to that conversation. That second clause is what makes the two layers feel like one feature. A chat whose most recent message is "see you tomorrow" still appears when your query matches something from three months ago — the preview would never have surfaced it.

## Opening a hit exactly where it happened

Tapping a message result does not just open the conversation. It opens it **at that message**.

The hit carries an initial message id into the chat screen. After the controller finishes its normal initialization, the view tries to scroll to that id. If the message is not in the first page the chat loaded, it fetches a context window around it first, then scrolls and highlights it. Most chats open pinned to the newest message; this is the one path that deliberately opens in the middle of history, and it has to work when the target is far from the bottom.

The point is not the animation but the position: you land with the matching line in view and the surrounding messages above and below, which is usually what you need to remember why you were looking.

## One search box, one visible tab

The chats list has tabs, and each tab keeps its own membership. Search respects that. Both the local rows and the message hits are scoped to the tab you are looking at.

That is a deliberate constraint rather than an oversight. Merging every tab into one result list would produce hits you cannot act on without first guessing which tab owns them. Scoping keeps the mental model simple: search narrows the list in front of you. If the conversation lives in another tab, switch tabs and search there.

The same scoping rule keeps a message hit from resurrecting a chat that the current tab would otherwise hide. A hit can keep its own chat row visible, but only when that row belongs to the visible tab in the first place. The new interface strings ship in all six supported locales, so the two sections and the empty state read the same way everywhere the app does.

## Escape the wildcards

The message search uses a `LIKE`-style substring match, and that syntax has two characters with special meaning: `%` matches any run of characters, and `_` matches any single one. Left alone, a query like `100%` or `snake_case` stops being a literal search and starts behaving like a pattern.

We added a small escaping step before the query reaches the database, and declared the escape character on the pattern itself:

```sql
content ILIKE :pattern ESCAPE '\'
```

The query's `%`, `_` and the escape character are escaped first, so the only wildcards in the final pattern are the ones the server adds around the search term. We applied the same treatment to the existing per-conversation message search, which had the same flaw. It is a small fix with a visible payoff: searching for a literal percent sign finds the message that contains one.

## What this does not do yet

Two limits are worth stating plainly.

**Encrypted chats are not message-searchable.** The server only ever sees ciphertext there, so it cannot match content it cannot read. Those conversations still match on identity and on their last-message preview, but they will never contribute a Messages hit. Searching the decrypted cache on the device is the eventual path; it is not in this version.

**Results are capped and ranked by recency.** The default cap is 50, and ordering is newest first. This is a substring search, not a tokenized full-text index, so it does not rank by relevance or understand word stems. For the common case — "I know roughly what was said, show me the recent candidates" — recency is the right default and the cheapest thing to reason about.

## FAQ

**Does search find messages older than the preview in the chats list?**
Yes. The preview only reflects the newest message. The Messages section comes from a server search over message history, so an old line can surface a conversation whose preview says nothing about your query.

**Why do encrypted chats never appear in the Messages section?**
Because the server stores and serves ciphertext for those conversations. It can match the chat's identity and its encrypted preview locally, but it has no readable content to search. That is a property of the encryption model, not a missing index.

**Is the search case-sensitive, and can I search for a literal `%`?**
It is case-insensitive. `%` and `_` are escaped before the query runs, so they are treated as ordinary characters rather than wildcards.

— the GwinG team
