---
layout: post
title: "Reserved names: keeping your app's brand out of a user's handle"
description: "How a group name became an impersonation vector, and the shared normalisation module that now protects handles, group names and the official channel."
tags: showdev, python, fastapi, webdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-08-reserved-names-official-channel.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-08-reserved-names-official-channel.png
---

A name in a notification is a claim about who is talking. When the row says **Folcadi Support**, most people read it as the app speaking, not as a stranger who picked a good title. That is the whole attack: cheap to set up, impossible to explain afterwards.

This is the story of closing that gap in Folcadi, a language-exchange messenger. It covers the three holes that made impersonation possible, the one shared module that now closes them, and the design decision that followed: a reserved name *is* the official channel, and an official channel behaves differently from a normal group.

## Three ways to look official

The app already had a reserved-name list for user handles. Group names had none. That asymmetry was the bug.

Three separate holes, each enough on its own:

1. **Group names were unprotected.** Anyone could create a group called "Folcadi" or "Folcadi Support". It would appear in other people's chat lists, and the invite push would read "Added to group: Folcadi". A group title travels further than a handle, because it shows up in someone else's notification.
2. **The handle rule lived only in the client.** The app limited nicknames to 2–20 characters of `[A-Za-z0-9_]`. A direct request to the API was not bound by a form validator, so it could store anything the database accepted.
3. **Matching was exact and untrimmed.** `"folcadi "` with a trailing space renders identically to `folcadi` but failed an exact-match comparison. The check was correct in the letter and useless in practice.

Any of the three makes the other two look like details. Fixing one at a time would have left the same outcome reachable.

## One module, two call sites

The fix starts with a single module that owns the rules: the reserved sets, the brand tokens, the normaliser and the nickname pattern. It has no database access and imports nothing from the feature layer, so both the profile path and the group path can use it without a cycle.

```python
RESERVED_NICKNAMES = frozenset({
    "folcadi", "folcadi-admin", "folcadi_official", "folcad-i",
    "admin", "support", "moderator", "system",
    # retired brands stay reserved so nobody impersonates them either
    "tongue", "tongue-admin", "tongue_official",
    "gwing", "gwing-admin", "gwing_official",
})

RESERVED_GROUP_NAMES = frozenset({
    "folcadi", "admin", "support", "moderator", "system",
})

BRAND_TOKENS = ("folcadi",)
NICKNAME_PATTERN = re.compile(r"^[a-zA-Z0-9_]{2,20}$")
```

The important part is not the list. It is that there is exactly one list. Two copies of a security rule drift on schedule: someone adds a variant to one path during a review and forgets the other, and the hole reopens silently. A shared module turns "did we update both?" into a compiler question.

## Normalise before you compare

Comparing user-supplied names is a normalisation problem first and a lookup problem second. The helper is deliberately boring:

```python
def normalize_name(value: str | None) -> str:
    """Lowercase + collapse whitespace, closing the `"folcadi "` bypass."""
    return " ".join((value or "").split()).lower()
```

Lowercasing handles `Folcadi`. Collapsing whitespace handles both `" folcadi"` and `"folcadi "`, plus the internal-tab variant that looks like a space. `.split()` with no argument also drops the empty strings, so the join is the normalised form.

Two matching strategies sit on top of it, and the difference is intentional:

- **Exact match** against `RESERVED_NICKNAMES` and `RESERVED_GROUP_NAMES`.
- **Substring match** against `BRAND_TOKENS` — currently just the live brand name.

The substring rule is the strict one: it also blocks "Folcadi fans" and "Folcadi Official". That is deliberate while there is no in-app badge that distinguishes a real official account from a convincing one. Over-blocking a few creative group names is cheaper than under-blocking one impersonation.

Retired brands are exact-match only. This is easy to get wrong. A messenger for language learners needs to let someone name a group **"Tongue twisters"**; only the retired brand *as a name* is off-limits. The interesting rule lives in the comment next to the list, not in the code, because that is where the next person will change it.

## The server is the boundary, not the form

The nickname rule moved into a validation function used by every path that can write a nickname: the user's own profile edit, the admin edit, and the availability check behind the sign-up form.

```python
def validate_nickname(nickname: str) -> str:
    value = (nickname or "").strip()
    if not is_valid_nickname(value):
        raise HTTPException(status_code=400, detail=NICKNAME_RULE)
    if is_reserved_nickname(value):
        raise HTTPException(status_code=400, detail="Nickname already taken")
    return value
```

Two response details matter more than they look.

**Invalid shape is a 400; reserved is "already taken".** The first tells the user to fix their input. The second should not confirm that `admin` exists as a reserved slot — from the outside, "reserved" and "someone has it" are indistinguishable, and that is fine. Different status codes, one message for the outside world.

**The stored value is the stripped value.** Returning the normalised result instead of validating a copy means the database only ever sees the shape that was checked. A validator that returns the original string is a subtle bug: it verifies `"  frank  "` and then stores the spaces.

Signup is the third writer, and it generates handles instead of asking for them. Generated names are truncated to fit the same pattern, so an account created through a provider never starts life with a handle it is not allowed to keep.

Group names got the same treatment one layer earlier. The request schema trims the name **before** the length validators run:

```python
@field_validator("name", mode="before")
@classmethod
def _trim_name(cls, value: object) -> object:
    return value.strip() if isinstance(value, str) else value
```

Without that ordering, a whitespace-only name passes `min_length=1`, and `" Folcadi "` reaches the reserved check unnormalised. Validator order is part of the rule.

## A reserved name is the official group

Once group names could not be taken by users, a product question appeared: what should *somebody* use the name for?

The app needed an official announcement channel — release notes, downtime notices, the things you want every account to see once. That needs a way to identify it. The tempting answer is a new column: `is_official`, a migration, an admin toggle, and a permanently new concept in every query that reads a conversation.

The cheaper answer was already in the schema. A reserved name is a marker nobody else can set. Staff can create or rename a group to a reserved name; everyone else gets a 403. So the name itself identifies the channel:

```python
def is_official_group(conv: Conversation | None) -> bool:
    return bool(
        conv is not None
        and conv.type == "group"
        and is_reserved_group_name(conv.name)
    )
```

No column, no backfill, no second source of truth. The trade-off is honest: renaming the channel requires staff, and the behaviour is coupled to the name. But the invariant is enforced in the same module that protects the brand, so the coupling cannot drift either.

Non-staff members cannot rename the group, transfer ownership, or dissolve it. Those are checked at the service layer, not just hidden in the interface.

## A channel you cannot silence

For a normal group, the user's controls are the point. Mute it, block the owner, leave it. For an announcement channel, the same controls mean a user can quietly opt out of the one channel that carries "we are down for maintenance".

So the official group changes three behaviours:

**Muting is refused.** Setting a mute on a reserved-named group returns 403. Unmuting is still allowed, because a mute that predates the rule should be recoverable. Quiet hours still apply — that is a device-level setting about when you want to be disturbed, not a per-conversation decision, and respecting it is the difference between "cannot be silenced" and "wakes you at 3 a.m.".

**Blocking has no effect.** Group conversations already skipped block checks, and that stays true: blocking the owner of the channel does not stop announcements.

**"Delete chat" only hides it.** This is the interesting one. Leaving a normal group deletes the membership row. Leaving the official group clears the member's reads, drafts, scheduled messages, private pins and unread rows, then sets a flag:

```python
if is_official_group(conv):
    member.is_hidden = True
    await db.commit()
    # ...and the row stays in the database
    return
```

The chat disappears from the list. The next message to the channel brings it back, because the broadcast path clears the flag before fan-out:

```python
update(ConversationMember)
.where(
    ConversationMember.conversation_id == conversation_id,
    ConversationMember.is_hidden.is_(True),
)
.values(is_hidden=False)
```

Normal groups are untouched: leaving still deletes the membership. The asymmetry is the feature. "Delete chat" is a statement about the list you look at, not about the relationship you have with the team — and for the channel that carries service notices, the relationship has to survive it.

One detail makes this work without new client code. The app already had a "message for an unknown group → reload the list" path. Setting the flag back to false server-side means the client's existing reaction re-renders the row. A server-side restore that needs no new client state is worth designing for.

## What this does not solve

Three limits are worth stating plainly, because a post about impersonation defence that claims completeness is the problem it describes.

**Homoglyphs are still possible.** Names are compared case-insensitively and whitespace-normalised, but not Unicode-normalised. A Cyrillic `а` in `folcаdi` looks identical in most fonts and is a different string. Real confusable detection is a separate project with its own data (mixed-script detection, skeleton mapping, a font-aware UI decision). Pretending the current normaliser covers it would be worse than saying it does not.

**Substring blocking has a false-positive cost.** "Folcadi fans" is a natural group name and it is refused. That is the price of not having an official badge. When a badge exists, the rule can loosen. Until then, the product chooses the impersonation risk over the naming-convenience risk, and that is a judgment call, not a technical one.

**A reserved list is not a trust system.** It stops the cheap attack — pick the brand as your name and wait for someone to misread it. It does not stop a compromised device, a lookalike outside the app, or a user who believes a screenshot. Reserved names reduce the cost of being fooled for the laziest case.

## What the tests pin

Two suites cover the rules, and the interesting assertions are the negatives:

- `"folcadi"`, `"Folcadi"`, `" Folcadi "`, `"FOLCADI"` are all rejected.
- `"folcadi "` with a trailing space is rejected even though it is a different string than the one on the list.
- `"Tongue twisters"` is accepted; `"tongue"` is not.
- A non-staff member creating or renaming to a reserved group name gets 403.
- The official group refuses a mute, but accepts an unmute.
- Leaving the official group keeps the membership and sets the hidden flag; the next message clears it.
- Leaving a normal group deletes the membership, as before.

That last pair is the one that catches regressions. A refactor that unifies the two leave paths would make all the positive tests pass and quietly break the official channel.

## The pattern

Strip away the app-specific parts and the shape is:

1. **One module owns every name rule** — reserved sets, normaliser, pattern, helpers. Two call sites, one source.
2. **Normalise before comparing** — case and whitespace at minimum, and say out loud what you are not normalising.
3. **Enforce on the server**, with the client as a convenience. A form validator is not a boundary; the API is.
4. **Let the reserved name carry the identity** when it is already unique and staff-only. Fewer concepts, fewer migrations.
5. **Design the exception to the user's controls deliberately**, and document why the channel that carries service notices behaves differently from a chat with a friend.

None of that is novel. The failure mode it prevents is boring too, which is the point: a name is a claim, and the app should decide which claims are available to anyone.

## FAQ

### Why not just show a verified badge?

A badge is the right long-term answer, and it needs a server-side source of truth first. Reserved names are that source of truth, and the badge renders it. Building the badge first would have meant a visual marker with nothing behind it.

### Does this change normal group names?

Only when the name contains the live brand token or matches the reserved list. `"Language practice"`, `"Book club"`, and `"Tongue twisters"` all pass. If a group name is refused, the reason is the 403 detail, not a silent rename.

### What happens to a group that already had a reserved name?

The rule is checked when a group is created or renamed. An existing group keeps its name until someone edits it; editing it then requires staff. The check is at the write boundary, which is where it can be enforced cheaply.

### Why is unmuting still allowed?

Because a mute can predate the rule. Refusing both directions would make a legacy mute permanent, and "we cannot notify you and you cannot fix it" is not a policy — it is a stuck state.
