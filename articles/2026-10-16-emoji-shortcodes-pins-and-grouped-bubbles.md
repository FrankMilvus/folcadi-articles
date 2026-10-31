---
layout: post
title: Inline emoji shortcodes, private pins, and grouped bubbles
description: "What shipped in a Flutter chat app this week — inline :emoji shortcode autocomplete, per-user pins, grouped bubbles, and the defects a review pass caught."
tags: flutter, showdev, tongues, languageexchange
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-16-emoji-shortcodes-pins-and-grouped-bubbles.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-16-emoji-shortcodes-pins-and-grouped-bubbles.png
---

This week in **GwinG**, a Flutter chat app for language exchange backed by a
FastAPI + PostgreSQL server, we shipped a batch of chat work: an inline
**:shortcode emoji autocomplete** in the composer, a per-user **"pin for me"**
that sits alongside the existing conversation-wide pin, and **sender grouping**
for message bubbles so a burst of messages from one person reads as a block.

We also ran a behavioural review pass over the whole batch before shipping it.
That pass is the interesting part: the tests were green and the analyzer was
clean, yet it caught three defects that would have shipped silently. I'll show
those at the end, because they're the reason we do the pass at all.

## Inline emoji shortcodes: `:joy` → 😂

People type `:joy`, `:thumbsup`, `:tada` — the aliases that grew up around chat
apps — not the formal Unicode names. So the composer now watches the token under
the caret and offers matching emoji in a horizontal strip just above the input.
Tap a cell, or press Enter, and the token is replaced with the glyph.

The table is GitHub's gemoji set — **1,913 aliases** — bundled as a JSON asset
so matching works offline. That's deliberate: the familiar shorthand (`joy`,
`thumbsup`, `tada`) simply isn't in Unicode's own names, so a pure CLDR-prefix
lookup would miss exactly the shortcodes people reach for.

### A token scanner that stays out of the way

The hard part of inline autocomplete is not the lookup — it's deciding *when*
there is a token at all. A scanner that fires too eagerly pops the strip while
you type a URL or a clock time. The rule set we settled on is deliberately
conservative:

- the caret must be **collapsed** — a range selection has no single token;
- the token is `:` followed by **at least one** shortcode character before the
  caret, so a bare `:` or a trailing space ends it;
- the `:` must **start the text or follow whitespace**, which is what keeps
  `https://…` and `10:30` from ever triggering;
- the run stops at the first non-shortcode character (`a–z`, `0–9`, `_`, `+`,
  `-`), so `:+1`, `:-1` and `:t-rex` all survive.

Scanning backwards from the caret over ASCII-only characters has a nice side
effect: the scan can never land in the middle of a surrogate pair, so multi-unit
emoji in the text around the caret can't produce a bad slice.

### Ranking so `:joy` beats `:joy_cat`

A plain "everything that starts with the query" list buries the obvious answer.
Ranking runs in three tiers, and each later tier only fills space the earlier
ones left:

```dart
// 1. exact shortcode first  →  :joy gives 😂 before anything else
final exact = shortcodes[query];
if (exact != null) add(exact);

// 2. then shortcode prefixes, shortest first
//    :smi → :smile before :smiley_cat
prefixes.sort((a, b) {
  final byLength = a.length.compareTo(b.length);
  return byLength != 0 ? byLength : a.compareTo(b);
});

// 3. finally gemoji tags, purely to widen recall
//    :happy, :workout — never ahead of a real shortcode
```

That third tier is what makes `:happy` find 😀 even though `happy` isn't an
alias — tags are discovery-only and never outrank a real shortcode. Emoji are
de-duplicated across all three tiers, and a query longer than 32 characters is
dropped so a runaway `:`-run costs nothing.

Accepting a suggestion replaces exactly the typed range with the glyph plus a
trailing space, and puts the caret after it — the same manual-insert path the
emoji picker uses, which is why setting the text can't re-trigger the strip and
loop.

## Private pins, next to conversation-wide pins

GwinG already had one pin per conversation — the banner at the top of the
chat that everyone sees. The new twist is a second, **private** pin that only
you see, synced across your own devices. The two coexist on the same message.

### The data model

Both live in one join table, distinguished by a nullable `user_id`:

| Column | Meaning |
|---|---|
| `conversation_id`, `message_id` | which message is pinned |
| `pinned_by` | who cast the pin |
| `user_id` | `NULL` = pin "for all"; a user id = that user's private pin |

Two **partial unique indexes** keep it honest — one "for all" pin per message,
and one private pin per user per message:

```sql
-- one shared pin, and one private pin per user, for a given message
UNIQUE (conversation_id, message_id) WHERE user_id IS NULL;
UNIQUE (conversation_id, message_id, user_id) WHERE user_id IS NOT NULL;
```

### One service, three delivery paths

Pins can arrive over the WebSocket, over REST, or be torn down when the message
itself is deleted. Rather than let three code paths re-implement the rules —
membership, who may pin for everyone, whether the message is deleted — they all
delegate to a single `pin_message` / `unpin_message` service. The rules themselves
are small but they matter:

- **"For all"** in a group is admin/owner only, and announces itself with a
  system row ("X pinned a message"). In a 1:1 chat both members may do it.
- **"For me"** is open to *any* member — even a non-admin in a group — and it
  never emits a system row or notifies anyone. The event is echoed **only to the
  actor's own devices**.
- Unpinning one scope leaves the other intact, so the client tracks the two flags
  separately (`pinnedForAll` / `pinnedForMe`) and OR-merges them on every update
  instead of clobbering.

The `scope` field defaults to `"all"`, so a client that predates the feature
keeps working unchanged.

## Grouped bubbles: one burst, one block

When the same person sends several messages in a row, they shouldn't look like
several separate conversations. Sender grouping squares the corners that touch a
neighbour and keeps the outer corners rounded, so a run reads as a single block.

The subtlety is that we compute **two different things** and they must not be
conflated:

- a **gap/name group** — tight spacing and no repeated sender label;
- a **corner run** — which corners get squared.

```dart
// The rectangle's sender-side corners follow the group position:
// first keeps its top, last keeps its bottom, a middle bubble squares both.
final radius = groupedMessageBubbleRadius(
  base,
  isMe: isMe,
  isFirstInGroup: !glueAbove,
  isLastInGroup: !glueBelow,
);
```

A corner run breaks on a sender change, a day rollover, a system chip, or a
**photo-album grid** — the grid always keeps its own fully-rounded shape and
never joins a run. Keeping the two notions separate is what lets a shortcode
strip, a date separator and an album all sit in the run without producing
mismatched seams.

## The defects a green build didn't catch

Before shipping the batch we run a behavioural review: read the diff end to end,
trace each event and field across the client/server boundary, and hand-verify
every candidate. The compiler and the unit tests were already happy, and it
still found three real bugs.

**1. A pinned encrypted message previewed as ciphertext.** The `message_pinned`
broadcast was hand-built in two places — the WebSocket handler and the REST
route — and both forgot the `is_encrypted` field that the canonical serializer
emits and the client reads. In an encrypted chat that meant the pin banner
showed the raw ciphertext instead of "🔒 Encrypted message". The fix wasn't to
add the field twice; it was to delete the duplication and build the payload in
**one shared function** so the two paths can't drift again.

**2. A half-glued bubble seam.** The grouping code decided "is this neighbour an
album?" from `albumId != null`, but an album is only *rendered* as a grid when
its run is longer than one message. A lone album row — its siblings outside the
loaded window — painted as an ordinary bubble while the glue logic still refused
to join it, so one side squared a corner and the neighbour stayed rounded: a
visible notch. The fix makes the glue consult the **rendered** album, not the raw
field, with a test that fails against the old behaviour.

**3. A floating button covering the last row.** The list reserved bottom padding
for its FAB stack sized against a **40dp** mini button. But a mini floating
action button lays out at **48dp** on mobile — the 40dp circle sits inside
Material's 48dp minimum tap target. On an admin account the extra 24dp pushed the
topmost button over the last row's timestamp. The fix derives the clearance from
the real box.

None of these threw an exception. All three built, ran, passed CI, and did the
wrong thing. That's the whole argument for reviewing behaviour, not just results.

## Why write it up

Each of these is small on its own. Together they're the difference between a chat
app that works and one that feels considered — the shortcode that completes
before you finish typing, the pin that's yours alone, the burst of messages that
reads as one thought. We write them up as we ship them so the polish gets its
share of the credit.

## FAQ

**Does emoji shortcode autocomplete work offline?**
Yes. The shortcode table ships as a bundled asset and is loaded once, so matching
never needs the network. If the asset ever fails to load, autocomplete simply
stays inert rather than breaking the composer.

**Can one message be pinned both privately and for everyone?**
Yes. They're independent records in the same table, and the client keeps two
flags per message so unpinning one leaves the other in place.

**Who can pin a message for the whole group?**
Admins and the owner. Any member — admin or not — can always pin privately for
themselves.

**How is offset handling safe with emoji in the text?**
The token scanner only walks back over ASCII shortcode characters, so it stops at
the first non-ASCII unit and can never split a surrogate pair. Caret offsets and
`String.length` are both in UTF-16 units in Flutter, so the replacement range is
always valid.

— the GwinG team
