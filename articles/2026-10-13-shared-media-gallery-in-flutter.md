---
layout: post
title: A shared media gallery for chats and profiles
description: How we built a shared media gallery for chats — one conversation-scoped read for images, files and links, a read-only peer resolver, and E2EE exclusion.
tags: flutter, fastapi, webdev, showdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-shared-media-gallery-in-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-13-shared-media-gallery-in-flutter.png
---

Open a friend's profile — or a group's info screen — and you can now browse every
photo, video, document and link the two participants have exchanged. We call it the
**shared media gallery**, a familiar idea from other messengers, scoped tightly to
the conversation between you.

The important word there is *conversation-scoped*. The gallery is not a cross-chat
index of everything a user ever sent; it's the media belonging to one thread. On a
profile that thread is the 1:1 chat between you and the person; in group info it's
the group. One read serves both shapes, and the profile reaches it through a
**read-only peer resolver** that never creates a conversation as a side effect. It
shipped as Phase 1, and this post walks through the parts that were worth getting
right: the single conversation-scoped read, its kind slices, cursor pagination, the
resolver, a small FastAPI routing gotcha, and why encrypted chats are deliberately
left out.

## Why scope the gallery to a conversation

The obvious alternative is a user-scoped media index: "everything this person has
ever shared, across all their chats." We didn't want that, for two reasons.

First, the feature's meaning is always conversational. When you open someone's
profile and tap **Media**, you're asking "what have the two of us shared?" When you
open group info, you're asking "what has this group shared?" Both questions are
about a single conversation, and answering them with a per-user index would force us
to reconstruct the conversation boundary in the query anyway.

Second, access control falls out for free. A conversation already has a
membership check on every read, so the gallery inherits it: if you're not a member,
you can't page through the media. No new sharing model, no new leak surface.

So the primitive is simple: given a conversation and a caller, return the messages
in that conversation that carry media.

## One read serves 1:1 and groups

The backend exposes a single service method, `fetch_shared_media`, and both entry
points delegate to it:

```python
@staticmethod
async def fetch_shared_media(
    db: AsyncSession,
    conversation_id: int,
    user_id: int,
    kind: str = "media",
    limit: int = 50,
    before: int | None = None,
):
    """Messages carrying media in a conversation (the "Shared Media"
    gallery shown on a profile and in group info)."""
```

It verifies the caller's access to the conversation, refuses encrypted threads (more
on that below), filters by kind, and returns newest-first. The same call powers a
1:1 profile tab and a group screen; there is no "group variant" and no "profile
variant" on the read path.

### The kind slices

A gallery isn't one list — it's three, and the frontend shows them as segments. The
`kind` argument selects the slice:

- **`image`** — messages with an image attached.
- **`video`** — messages with a video attached.
- **`media`** (the default) — images *and* videos, which is the photos grid.
- **`file`** — documents, i.e. messages carrying a file attachment.
- **`link`** — messages the backend attached a link preview to.

The filters are plain column checks, which keeps the slice logic in one place and
easy to read:

```python
if kind == "image":
    kind_filter = Message.image_url.is_not(None)
elif kind == "video":
    kind_filter = Message.video_url.is_not(None)
elif kind == "file":
    kind_filter = Message.file_url.is_not(None)
elif kind == "link":
    kind_filter = Message.link_preview_url.is_not(None)
else:  # "media" (default): images and videos
    kind_filter = or_(
        Message.image_url.is_not(None),
        Message.video_url.is_not(None),
    )
```

`link` is worth a note: it doesn't mean "message text contains a URL." It means the
backend unfurled the URL and stored a preview, which is exactly what the link list
renders.

### Deleted messages, ordering, and the cursor

Two invariants keep the list honest. Deleted messages are excluded outright —
`deleted_at IS NULL` is part of the `WHERE`, so a removed photo doesn't linger in
the grid. And results are ordered `Message.id.desc()`, newest first, matching how
every other list in the app reads.

Pagination is a keyset cursor, not an offset: pass `before`, the id of the oldest
message you've already seen, and the query adds `Message.id < before`. That stays
correct while new messages arrive — an offset would shift rows out from under the
reader as the conversation grows. The route takes the same three knobs, so the
client can fetch the first page and then ask for older media on demand.

There's one subtle, very real backend detail here. The response serializes the
*reply* sender's name, so `reply_to.sender` is eager-loaded on the query. A lazy
load inside an async session raises `MissingGreenlet` at serialization time — the
same bug family we hit on search. It's the kind of thing that only shows up when a
gallery item happens to be a reply to an image, which makes it exactly the kind of
thing to pin down in a query rather than discover in production.

## A read-only peer resolver that never creates a conversation

The profile screen knows a *user* id, not a conversation id. Normally "start a chat"
means get-or-create: look for the conversation, and if it isn't there, make one.
That's the wrong contract for a profile tab. Opening someone's profile is a *read*,
and it shouldn't quietly create an empty conversation just because they glanced at a
Media tab.

So the peer path is a dedicated resolver, `fetch_shared_media_with_user`, and it
resolves without ever writing:

```python
low, high = (user_id, peer_id) if user_id <= peer_id else (peer_id, user_id)
stmt = (
    select(Conversation.id)
    .where(
        Conversation.type == "private",
        Conversation.is_encrypted.is_(False),
        Conversation.user_low == low,
        Conversation.user_high == high,
    )
    .order_by(Conversation.id.asc())
    .limit(1)
)
conversation_id = (await db.execute(stmt)).scalars().first()
if conversation_id is None:
    return []
return await ChatService.fetch_shared_media(db, conversation_id, user_id, ...)
```

The pair is normalized to an ordered `(low, high)` so both directions of the same
1:1 point at one row. The lookup is restricted to a *plaintext private* conversation
and returns an empty list when there isn't one — never a new row. From there it
delegates to the same `fetch_shared_media`, so the resolver only ever has to know how
to find a conversation, not how to read its media.

That "returns empty when none exists" behaviour is what makes the read idempotent:
call it once or a hundred times, the state of the database is identical.

## Two routes, and the literal-first ordering gotcha

The API exposes two endpoints:

```http
GET /chat/with/{user_id}/media?kind=&limit=&before=
GET /chat/{conversation_id}/media?kind=&limit=&before=
```

The first is for profiles (peer id), the second for groups and chat info
(conversation id). Both map onto the service methods above.

Here's the gotcha, and it's a genuine one. The second route's path segment is
`{conversation_id}`, a placeholder that will happily match *any* single segment —
including the literal string `with`. If it were declared first, a request to
`/chat/with/123/media` would match that placeholder, FastAPI would then try to
validate `with` as an integer, and the client would get a `422` rather than the
profile gallery. FastAPI resolves routes in declaration order and validates the path
parameter's type *after* a route matches, so a later literal route is simply never
reached. The rule that follows is boring and effective: **declare the literal,
more-specific route first.** We do, and the `with` segment can never be mistaken for
a conversation id. The FastAPI docs on [path parameters and their
order](https://fastapi.tiangolo.com/tutorial/path-params/) are worth a read if you
haven't been bitten by this yet.

## Encrypted chats are deliberately excluded

We ship opt-in end-to-end encrypted chats. Their media is encrypted on the client
before it ever reaches the server, which means the server is holding ciphertext — and
ciphertext can't be filtered or listed. There is no "is this an image?" column to
query, because the image is inside a blob the server can't read.

So the feature is honest about it in two different places:

- The **conversation-scoped** read checks whether the conversation is encrypted and
  returns **403** if it is — "Shared media is not available in encrypted chats." An
  explicit refusal beats a silently empty list that looks like a bug.
- The **peer resolver** only ever looks at the plaintext private conversation, so a
  profile Media tab for someone you only share an encrypted chat with simply returns
  nothing. It doesn't 403, because from the resolver's point of view the two of you
  have no plaintext conversation — which is true.

Making this work later means decrypting from the local cache on the client, a
separate effort we've noted rather than bolted on. Some things are worth doing once,
properly.

## The frontend: Media / Files / Links

The Flutter side is a single reusable widget, `SharedMediaView`. It takes *exactly
one* of `userId` (profile path) or `conversationId` (group path), asserted at
construction, and stores three lazy segment caches so switching tabs doesn't re-fetch
what you've already seen.

The segments are Media, Files and Links, switched with Material's
[`SegmentedButton`](https://api.flutter.dev/flutter/material/SegmentedButton-class.html),
and the Media segment renders a 3-up grid that reuses the pieces the rest of the app
already has: `CachedNetworkImage` backed
by the shared `AppMediaCacheManager` for cached thumbnails, and the existing
fullscreen gallery for the tap-through, so the gallery you get here behaves exactly
like the one in a chat. Videos render a play-button placeholder in the grid and open
in the same viewer.

Files and links are lists. A file row shows the file name and a human-readable size
and opens in the browser on tap. A link row leads with the unfurled thumbnail, then
the preview title, with the site name underneath — the same data the link-preview
message bubble uses, which is why the `link` slice keys off the stored preview rather
than parsing text.

For entry points that aren't already a tab — pushing the gallery from group info — a
thin `SharedMediaScreen` hosts the widget full-screen. The profile reuses its tab
framework, adding an optional **Media** tab scoped by `userId`.

If you like the media plumbing behind this, two sibling posts go deeper: [photo
albums in
Flutter](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-photo-albums-in-flutter.html)
covers the multi-image layout, and [the Flutter Linux desktop video and media
kit](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-flutter-linux-desktop-video-media-kit.html)
covers playback. The [session durability work with the Linux
keyring](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-linux-keyring-session-durability.html)
is the other half of getting desktop right, and there's a [full overview of
GwinG](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html)
if you're new here.

## FAQ

**Does opening someone's profile create a chat?**
No. The profile path goes through the read-only peer resolver, which returns an empty
list when the two of you have no plaintext conversation. It never writes a row.

**What happens on an encrypted chat?**
The conversation-scoped endpoint returns **403** — an explicit "not available here."
The profile path returns an empty list, because the resolver only consults the
plaintext 1:1 conversation.

**Can the gallery show media from a group and a 1:1 the same way?**
Yes. Both are the same conversation-scoped read; the only difference is whether the
client passes a `conversationId` or resolves one from a `userId`.

**Do deleted messages still show up?**
No. Deleted messages are excluded at the query level, so a removed photo disappears
from the grid on the next load.

**Can I page further back through old media?**
The API is cursor-paginated by message id via `before`, newest-first. The Phase 1 UI
loads the first page; wiring the grid's scroll to pull older pages is a small
follow-up on the same cursor.

— the GwinG team
