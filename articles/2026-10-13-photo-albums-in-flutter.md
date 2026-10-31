---
layout: post
title: Building photo albums in Flutter
description: "Building photo albums in Flutter: how GwinG layers an album group key over a one-row-per-media model, with an aspect-aware grid."
tags: flutter, dart, ux, showdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-photo-albums-in-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-13-photo-albums-in-flutter.png
---

Multi-select a handful of photos in **GwinG** and they land as a single grid
bubble — one caption, one timestamp, one tick, one notification — instead of a
full-size bubble per image. That's the **album bubble**, and it's the feature we
shipped this week on top of a messenger that, until now, was built entirely around
one image per message.

The interesting part was never the grid. It was doing this **without** tearing up
the invariant the whole codebase leans on: *one message row = one media item*. The
rest of this post is how we layered Flutter photo albums on top of that rule
instead of around it.

GwinG is a language-exchange messenger — chat, calls and live translation,
Flutter on the client, FastAPI and PostgreSQL behind it. If you're new here, start
with the [overview of GwinG](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html).
This post is one of a few we're writing today, alongside the
[shared media gallery](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-shared-media-gallery-in-flutter.html)
and the [Linux desktop video work](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-flutter-linux-desktop-video-media-kit.html).

## One row per media item — the rule we refused to break

Here's the constraint, stated plainly. Our `messages` table is one row per piece
of media. That single fact is load-bearing across the app:

- **Delivery reconcile is URL-based.** When the upload pipeline finishes, the
  client matches the optimistic bubble to the real row by the uploaded file's URL
  — one file, one row, one match.
- **Read receipts and reactions are per row.** A thumb, a "read" tick, a count —
  all column-level facts on a message.
- **The offline outbox is one durable record per file.** Every attachment gets its
  own record with its own progress and retry, so a killed app can resume each file
  independently.
- **Orphan cleanup scans `image_url` per row.** A row that never got its URL is an
  orphan; the sweep already knows how to find it.

A naive album feature — "add an `attachments` table" or "stuff a JSON array of
URLs into one message" — breaks all four at once. Every read path (history,
search, pins, previews) would need rewriting, and the URL-based reconcile would
stop matching because one row would now carry N URLs.

So we didn't do that. We kept one row per media item and layered a *group key* on
top of it.

## Why Flutter photo albums need a group key

Three nullable columns on `messages` turn N independent rows into one logical
album:

```sql
ALTER TABLE messages ADD COLUMN album_id    TEXT    NULL;
ALTER TABLE messages ADD COLUMN album_index INTEGER NULL;
ALTER TABLE messages ADD COLUMN album_size  INTEGER NULL;
CREATE INDEX ix_messages_album_id ON messages (album_id);
```

The [additive-column](https://www.postgresql.org/docs/current/ddl-alter.html)
shape matters: it's three plain columns and an index, not a new entity.

- **`album_id`** (indexed `TEXT`) is the shared group key. The *client* generates
  it at send time — a UUID-ish token — so an offline or still-uploading album is
  already grouped before any server round-trip. Its length is bounded on the
  server so a client can't stuff the indexed column with unbounded text.
- **`album_index`** (`INTEGER`, 0-based) is the **only ordering authority inside a
  group**. This one matters more than it sounds. Sibling rows upload concurrently,
  so their `created_at` timestamps can arrive out of order. If any code ever sorted
  a group by time, the grid would shuffle. It doesn't — it sorts by `album_index`,
  full stop.
- **`album_size`** (`INTEGER`) is the total tile count, duplicated onto every
  sibling on purpose. Denormalizing it means the `+N` overlay and the notification
  label render *before* the last sibling has uploaded, without a `COUNT(*)` on the
  hot path.

Because the columns are nullable and additive, the migration is safe to deploy
ahead of the client — and the part we care about most is that **`album_id IS NULL`
renders exactly like before**. Old rows, old clients, no regressions.

### The anchor row

An album needs *somewhere* to hang the things that only make sense once per group.
We call that row the **anchor**, and it's simply the sibling with the lowest index
(`album_index == 0`):

- it carries the **caption** (every other sibling stores an empty `content`);
- it carries the **reply** target, and therefore the quote preview and the
  "forwarded" flag;
- it owns the **footer surface** — time, ticks, read count — and the **reaction
  surface**;
- it's the row that fires the **notification**.

Siblings are collapsed: no caption, no reply, no footer, no separate push. The
server enforces the shape on send — `album_size` within range,
`0 <= album_index < album_size`, media present, and `album_id` rejected on a row
with no image — so a client can't post a half-formed group.

## Grouping Flutter photo albums on the client

Grouping happens on the reader's device, in the same pass that builds the message
list's display items. The rule is deliberately narrow: **consecutive** rows that
share an `album_id`, from the **same sender**, with no date separator or system
chip between them, collapse into one album item. Break the run — a different
sender replies, a day boundary lands — and you get two albums, which is correct.

Inside the item, the tiles are ordered by `album_index` and never by time:

```dart
siblings
  ..sort((a, b) => (a.albumIndex ?? 0).compareTo(b.albumIndex ?? 0));
```

That's the whole trick. There's no album table, no join, no second read path. The
list already built display groups for date separators and sender-name bands; an
album is just one more kind of group, and it counts as a **single visual unit**
for the gap logic between messages.

## A grid that's sized on the first frame

The premium feel of an album lives or dies on one thing: **the grid must be
exactly the right size before the images load.** Get that wrong and the whole
conversation jumps as tiles pop in.

We avoid the jump by sizing from data we already store. Every image row keeps its
`image_width` and `image_height` from the moment of upload, and the album block's
aspect ratio comes from the anchor's own dimensions (clamped so an extreme
panorama can't produce a degenerate grid). Each tile uses `BoxFit.cover`, so it
fills its cell whatever its native ratio — the same
[layout model](https://docs.flutter.dev/ui/layout) Flutter applies everywhere
else. Sender and recipient both get the final layout on frame one; the images
just fill in.

The layout is nothing more than a lookup from tile count:

| Count | Layout |
|---|---|
| 1 | single full bubble — the existing path, unchanged |
| 2 | two equal columns |
| 3 | one large tile left, two stacked right |
| 4 | 2×2 |
| 5 | two on top, three on the bottom |
| 6 | 3×2 |
| 7–9 | three columns; the last row is left-aligned |
| 10+ | capped at ten, with a frosted `+N` pill over the last tile |

Two small details make it feel finished. Tiles **cascade** in with a short
per-index fade-and-slide — and that stagger is gated on the app's motion setting,
so it's simply zero when the user has reduced motion on. And the whole grid sits
inside one shared media border, so only the *outer* corners are rounded and
accent-bordered; the interior seams are a hairline 2 px.

Open one tile and you land in the swipe-through gallery at exactly that photo.
That gallery is its own story — we wrote it up in the
[shared media gallery post](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-shared-media-gallery-in-flutter.html)
— and albums just hand it an entry point and a starting tile.

## One notification, one preview, one progress bar

An album is one *logical* message, so it should behave like one to the person
receiving it — and this is where the fan-out bugs live.

**Notifications.** Exactly one push per album, fired from the anchor. A ten-photo
album must not buzz ten times. The broadcast layer treats a sibling (any row with
`album_index > 0`) as suppressed: it still emits the message event so the tile
renders and gets its delivered tick, but it skips the push and the
conversation-list preview entirely. The body is the same everywhere — `📷 N photos`
— whether it's the push, the in-app notification list, or the friend-list preview.

**Uploading.** The send path is still per-file, which is what gives us per-tile
progress, retry and cancel for free. For an album we generate one `album_id`, then
loop the staged files, putting the caption and reply on the anchor only:

```dart
final albumId = newAlbumId();
final albumSize = staged.length;
for (var i = 0; i < albumSize; i++) {
  final isAnchor = i == 0;
  beginPendingUpload(
    filePath: staged[i].path,
    kind: UploadKind.image,
    content: isAnchor ? text : '',        // caption lives on the anchor
    replyToId: isAnchor ? replyId : null, // so does the reply
    albumId: albumId,
    albumIndex: i,
    albumSize: albumSize,
  );
}
```

On screen that becomes **one** thin aggregate bar across the bottom of the grid,
plus a small spinner on each tile still in flight. A tile whose upload fails gets
its own retry badge instead of spinning forever. Each tile is its own durable
outbox record — the same "this must survive a session ending" instinct behind our
[Linux keyring session work](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-linux-keyring-session-durability.html)
— so killing the app mid-send restores the album, grouped, and resumes the tiles
that hadn't landed.

## Forward keeps the group; delete takes it apart

Two actions are worth calling out because albums change their meaning.

**Forward** copies `album_id`, `album_index` and `album_size` onto every copy, so
the album **re-groups in the destination**. Forward ten photos and the other
conversation gets an album, not ten loose images.

**Delete** on an album removes **all** siblings. And there's a tidy edge case on
the other side: if an album ever drops to a single surviving row — siblings
deleted one by one, say — that row clears its `album_id` and **degrades back to a
plain single image**. The group key is optional, so the model simply falls back to
the behaviour it had before albums existed.

## What we'd do next

This phase is deliberately **images-only**. The model doesn't need to change to
add video tiles — `album_index` is media-agnostic — so mixed photo-and-video
albums are a clean later step: a per-tile player, a play badge, and the chunked
upload path video already has. The
[Linux desktop video pipeline](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-flutter-linux-desktop-video-media-kit.html)
is a good look at the machinery that would slot in. Encrypted chats stay as they
are: media is already blocked there, so albums inherit the same guard and the same
friendly message.

The takeaway, if there is one: a big feature doesn't have to mean a big schema. We
added three nullable columns, one convention (the anchor), and a client-side
grouping rule — and kept the invariants that make delivery, receipts, reactions
and cleanup work. The grid is the visible part. The three columns are the whole
trick.

## FAQ

**Do albums need a separate API endpoint?**
No. An album is just a normal send with `album_id`, `album_index` and `album_size`
attached to each row. The same REST and WebSocket send paths handle it, and the
same broadcast fans it out. There's nothing album-specific to route.

**How are album tiles ordered if the uploads finish out of order?**
By `album_index`, always. Sibling rows upload concurrently and can land with
timestamps in any order, which is exactly why we forbid `created_at` from ordering
a group. The index is the only authority.

**What happens if an album loses a sibling?**
The group stays a group while any sibling remains — a failed tile shows a retry
badge rather than a hole. If the album drops all the way to one row, that row
clears its `album_id` and renders as a normal single image.

**Does this work in encrypted chats?**
Not yet. Media is already disabled in encrypted chats, so albums inherit the same
restriction and the same user-facing message. When photo and video sending arrives
there, albums will come along for free.

— the GwinG team
