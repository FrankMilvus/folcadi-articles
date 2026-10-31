---
layout: post
title: Video thumbnails without downloading the video
description: How GwinG gives every video tile a first-frame poster the sender extracts at send time, so shared media previews without a download.
tags: showdev, flutter, video, webdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-18-video-thumbnails-without-downloading.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/video-thumbnails-without-downloading.png
---

## A video tile that only knew how to say "video"

Open the [shared-media view](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-shared-media-gallery-in-flutter.html)
in GwinG and you get three tabs: **Media**, **Files**, **Links**. Images have
always had thumbnails there. Videos had a play triangle on
a flat background and nothing else — the same tile for a five-second hello and a
ten-minute walk-through. The only way to tell two clips apart was to open one and
wait for it to start downloading.

That is the small annoyance we fixed this week. Every video sent from now on
carries a **poster**: a single JPEG of its first frame, extracted on the sender's
device and stored next to the message. The media grid shows that frame under a
small play badge, so you can recognise a clip before a single byte of video
crosses the network.

The rest of this post is about the choice behind it: where the frame comes from,
why we did not ask a server to make it, and the one desktop platform that needed
a different recipe.

## Two ways to get a still frame

There are exactly two honest options for a video poster, and they differ in *who*
does the work.

### Extract on the server

The classic approach. When a video finishes uploading, a worker runs a video tool
over it, grabs frame zero, stores the result, and attaches the URL. It is robust.
It works for every video in the database, including the ones sent years before
the feature existed, because a backfill job can walk the library and generate the
missing posters.

The bill is real, though. It puts a video decoder in the API image, spends server
CPU on every upload, and requires a schema and API change anyway. For a small
team, "add a media pipeline" is a project, not a line item.

### Extract on the sender's device

The alternative: the phone that is about to send the clip already has the file
and, in most cases, already has a decoded frame. Grab that frame before the
upload, encode it as a JPEG, and upload it as an ordinary image. The message then
points at both files — video and poster.

This is the familiar messenger pattern, and it has a pleasant property: the
expensive part happens once, on hardware that is already awake and already
holding the video.

## Why the sender's device won

Three reasons, in the order they mattered to us.

**The server could have done it, but it did not need to.** Shared media is
plaintext, and encrypted chats block media entirely, so a server-side extractor
would not have run into encrypted blobs. That removed the strongest argument
*for* server extraction. When both paths are technically possible, the cheaper
one wins.

**The code already existed.** The client has extracted first frames for
pending-upload placeholders for a while — the little preview that appears in a
bubble while a video is still in flight. Reusing that path meant the new feature
was mostly plumbing: take the frame we already know how to take, upload it, and
remember the URL.

**The send path already knows how to be resilient.** Uploads can fail, apps can
be killed, and a retry can happen hours later. A poster produced at send time
fits the outbox model instead of fighting it.

The trade-off is stated plainly: a poster ships with the send. Videos that were
already in the database when this landed do not have one, so their tiles keep the
old play-icon placeholder. That is not a bug in the renderer; it is the shape of
the decision. The follow-up list has a server-side backfill for exactly this
reason.

## How a poster travels from send to screen

### Extract, convert, upload

After the video upload finishes, and before the message is delivered, the upload
pipeline extracts frame zero, copies it into a `.jpg` so the server's
allowed-extension check is happy, and sends it through the normal image path. The
resulting URL is written onto the outbox record for that message.

The whole step is best-effort. If extraction fails, or the poster upload fails,
the message still goes out — just without a thumbnail. A missing poster is a
cosmetic loss; a blocked message is not. That is the right failure mode for chat.

### Survive a restart

The outbox is durable, and so is the poster URL. A restored send re-delivers the
same poster instead of extracting and uploading it a second time. On a retry, the
pipeline checks whether a prior attempt already produced one and skips the work.

This matters more than it sounds. Retries are the normal case on a bad
connection, not an edge case, and re-uploading a thumbnail on every attempt would
turn a cheap feature into a small denial-of-service against our own upload
endpoint.

### Render the tile

In the shared-media grid, each video tile draws the poster with a scrimmed play
badge over it. The list already knows how to lay out media tiles, including the
grouped tiles used by [photo albums](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-13-photo-albums-in-flutter.html).
No poster means the old placeholder — same layout, same tap target, no special
case in the list. The badge is the only thing that says "this
is a video"; everything else is just an image.

Because the poster is a normal image, it inherits all the boring guarantees. It
counts against upload quota, it ages out under the same TTL, and the orphan sweep
protects it like any other upload. Forwarding a video copies the poster with it,
so the tile looks right in the destination conversation too. When a group is
dissolved, the poster is deleted along with the rest of that group's media.

While we were in that view we also centered the **Media / Files / Links**
selector. It had been hugging the left edge on wide screens; now it centers when
it fits and still scrolls when the width is tight.

## The desktop plot twist: Linux needed its own extractor

The mobile path relies on two Flutter plugins that register Android and iOS only.
On desktop they throw an "unimplemented" error, the client catches it, and the
send quietly proceeds without a poster. That catch is why the feature looked fine
on phones and simply never appeared on a Linux desktop — and why the
pending-upload placeholder and the in-bubble preview were missing there too.

The fix branches the extraction helper on the platform. On Linux it opens the
file with a media_kit player, waits for the first decoded frame, and captures
that frame as a muted JPEG screenshot. No separate video-decoding stack, no
second player implementation — just a short-lived player used as a camera.

One subtlety: this capture deliberately does **not** take the serialization gate
that the chat and full-screen players use. Those players call the extraction
helper while they already hold that gate, so acquiring it again would stall the
exact thing we were trying to preview. The helper reads a frame; it does not
coordinate playback state.

The pleasant side effect is that the branch lives in the shared thumbnail helper,
not in the media grid. The pending placeholder and both video players get desktop
previews at the same time, from the same change.

## Old videos, Windows and macOS

Nothing changes for a video that has no poster. The renderer falls back to the
play icon, which is what those messages showed before. The honest fix for the
back catalogue is a server-side job that generates posters for existing rows, and
that is on the follow-up list, not in this release.

Windows and macOS also still show the icon. The desktop app ships media_kit
libraries for Linux only — desktop video is a Linux feature today — and neither
platform has a frame extractor wired up. Getting posters there means adding the
platform libraries and a capture path, and we would rather do that deliberately
than ship a half-tested branch.

## Tests and what comes next

The change is covered on both sides of the wire. On the backend, four tests
exercise a REST send round-trip, the nullable case, the WebSocket message
payload, and a forwarded copy that must keep its poster. On the client, six tests
cover JSON round-trips for messages and outbox records, both when a poster is
present and when it is absent. The absent case is the one that protects old
messages.

The follow-ups are small but worth naming:

- a server-side backfill for videos sent before this change;
- Windows and macOS posters, once those platforms have the video stack to match;
- using the poster in the chat bubble's loading state and in the full-screen
  player *before* the video downloads — the data is already there;
- a low-quality placeholder, so the tile paints something instantly while the
  real poster loads.

None of these change the shape of the feature. They fill it in.

## FAQ

**Does every video now upload two files?**

Yes — the video and a small JPEG. The poster is a normal image upload, subject to
the same quota and TTL rules as any other image. If it fails, the video is still
sent.

**Why not extract frames on the server and avoid the extra upload?**

We could have: the media is plaintext, so nothing blocked it. But that adds a
video decoder to the API image, spends server CPU on every upload, and still
needs schema and API changes. The client already knew how to capture a frame, so
the expensive work stays on the device that owns the video.

**Why do some older videos still show a play icon?**

Because posters ship with the send. A video that was uploaded before this change
has no poster stored, and the renderer deliberately falls back to the placeholder
rather than showing an empty tile. A backfill job is on the list.

— the GwinG team
