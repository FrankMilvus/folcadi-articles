---
layout: post
title: "A friend count you can switch off: owner-side privacy in a Flutter app"
description: "Friend count privacy is more than a toggle: a badge, a hidden flag, a friends list that has to go with it, and a setting that must reset per account."
tags: showdev, flutter, privacy, ux
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-12-friend-counts-privacy-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-12-friend-counts-privacy-flutter.png
---

A search result row is a small argument for talking to someone: a photo, a name, a city, the languages they know. We added one more claim to that row — how many friends the person has, drawn as a small icon and a number against the right edge.

The badge is a small change. Keeping the promise behind it is the larger one. Friend count privacy sounds like a toggle that turns a number into a zero, and it is not that at all: a count is a fact about a set, and the set has its own ways of being visible. The real work was in finding those ways before someone else did.

## The badge is the easy part

The row renders the icon and the number only when the number is positive. A user with no friends, a user who hid the count, and a user whose count failed to load all produce the same clean row. There is no viewer-side switch in the tile, because the server has already decided what this particular viewer is allowed to see.

```dart
// SearchResultTile: the server has already applied the owner's choice.
if (user.friendCount > 0)
  Row(
    mainAxisSize: MainAxisSize.min,
    children: [
      Icon(Icons.people_outline, size: 14),
      const SizedBox(width: 2),
      Text('${user.friendCount}'),
    ],
  ),
```

That little `if` encodes a decision worth stating out loud: on a search row, "no number to show" is the only distinction the UI needs. The client is not allowed to guess whether a missing badge means *none* or *private*. That question is answered on the profile, where there is room to answer it honestly.

## One number, computed once for a whole page

The canonical count is deliberately narrow. It is the number of distinct peers with an accepted friendship, never the user themselves, never a pending request, and never a peer with a block relationship in either direction. Distinct peers matter because a friendship is an edge that can be recorded from either side; counting rows instead of people drifts upward the moment the data has any duplication. A blocked peer is not a friend of anyone, so it does not inflate the number either.

The count is also computed for the page, not for the row. One query loads every accepted or blocked edge touching any of the requested users; the rule above is applied in memory; a second query loads the relevant owners' settings. A page of fifty results therefore costs the same as a page of one — two queries, plus a role check that only runs when something is actually hidden.

That last clause is the kind of detail that pays for itself. The role lookup is the extra step, and a page where nobody hid anything never pays for it.

## Friend count privacy is a list problem, not a number problem

The setting lives in Settings → Privacy, next to the presence and activity visibility choices, and it is on by default.

When it is off, the obvious part happens: the count comes back as zero, so no badge renders and the profile card has nothing to show. The non-obvious part is that the same profile has a friends list, and the list is served as public profiles. Its *length* is the count. Zeroing the number while still returning the full list would leak it in the most direct way available — not an estimate, not an inference, just the data with a different label.

So when the count is hidden, the public friends list returns nothing. The profile then has to be honest about why: the Friends card disappears from the overview tab, the tab label loses its number and reads plain "Friends" rather than "Friends (0)", and the Friends tab says the user keeps their friends private instead of rendering the standard empty state. "No friends yet" is a statement about a person. On a private profile, it would be a false one.

The same reasoning applies to every future surface. Any endpoint that returns a collection whose size equals a private number is a count endpoint wearing a different hat.

## Why zero is a lie: the `hidden` flag

A count response with just a number cannot express the two states that matter. `0` would mean both "this person has no friends" and "this person is not telling you", and the profile would have to pick one — which means the app would either claim something false about its user or show nothing at all and look broken.

The response carries both values instead:

```json
{ "count": 0, "hidden": true }
```

The number stays renderable, and the flag carries the reason. The profile reads the flag to decide between a friends card, a privacy note, and a normal empty state. The search row ignores the flag because it only needs the renderable number.

That split is the whole contract, and it is worth defending in review: the number is for display, the flag is for meaning. Merging them into a nullable field would work in Dart and would make every call site think about nullability instead of about disclosure.

## Owner and admin bypass — role, not plan

The owner always sees their own count, and staff accounts with an admin or owner role do too.

The bypass hangs off the role axis, not off a paid plan or a subscription flag. That choice is not about principle alone; it is about what stays stable. A plan-based bypass would silently change what users see every time pricing changes, and it would couple a privacy rule to a billing decision. A role-based bypass has exactly one question to answer — is this viewer staff? — and that answer does not move when the product does.

In the batched path the bypass is applied per page: if the viewer is an owner or staff, the hidden set is discarded wholesale before the counts are zeroed, and the response is the true numbers. For a single profile, the same rule is expressed as a visibility check that returns true for the owner and for staff, and consults the stored setting for everyone else.

## The viewer id has to travel with the request

The batched count is viewer-aware. Search, the random feed, and the friends appended to the initial feed all pass the viewer's id into the count call.

That parameter is easy to forget and expensive to omit. A batch function without a viewer can only produce one answer for the whole page, and that answer has to be either the public one — so staff lose their bypass — or the unfiltered one, which is far worse. Threading the viewer through keeps the query shape identical for everyone and moves the filtering to the last step, where it belongs.

```python
async def counts_for_page(db, user_ids, *, viewer_id):
    counts = await canonical_counts(db, user_ids)        # one query
    hidden = await owners_who_hid_their_count(db, counts)  # one more
    hidden.discard(viewer_id)
    if hidden and await is_staff(db, viewer_id):
        hidden.clear()
    return {uid: 0 if uid in hidden else n for uid, n in counts.items()}
```

Two properties fall out of this ordering. The hidden set is computed once for the page, and the role check is only reached when the set is still non-empty. A page where nothing is hidden never pays for it.

## The account-scope trap in a local-state settings switch

The switch is backed by a local notifier, because settings pages have to respond instantly and a round trip per tap is not acceptable. The value is synced to the stored profile settings on a short debounce, and read back from the server when a profile loads.

Local state outlives a session, and that is the trap. Sign out, sign in as someone else, and the notifier still holds the previous account's choice. The next settings save then writes that choice into the *new* account. A privacy setting that silently applies to the wrong person is worse than no setting at all, because the person it protects never chose it.

The fix is a reset to the default on sign-out, alongside the other account-scoped visibility choices. It is one line, and it is the line that makes the rest of the feature trustworthy.

There is a second, smaller trap in the same page: the server sync is debounced. Toggle the switch and immediately navigate away, and the choice may never leave the device. The next profile load then flips the switch back, which looks like the app ignoring the user. So the page flushes a pending sync when it is disposed:

```dart
@override
void dispose() {
  final pendingSync = _syncTimer?.isActive ?? false;
  _syncTimer?.cancel();
  if (pendingSync) unawaited(_syncSettings());
  super.dispose();
}
```

Neither of these is about the friend count specifically. Every account-scoped setting has the same two failure modes, and this feature just made them visible.

## The invariants we now test

The behaviour is written down as invariants rather than as screenshots, and the count side is pinned by tests:

| Invariant | What it means |
|---|---|
| Canonical count | Distinct accepted peers only; self, pending requests and blocked peers in either direction excluded |
| Hidden count | A hidden owner's count is zero for every non-owner, non-staff viewer |
| Bypass | The owner and staff still receive the real number |
| Withheld list | The public friends list is empty when the count is hidden |
| Response default | A user response built without a count still serializes zero, so unrelated endpoints keep working |

The last row is the quiet one. Adding a field to a shared user response is a small change with a wide blast radius: every endpoint that serializes a user but never runs the count query must still produce a valid value instead of failing or returning null. That one is pinned with a test, which is cheaper than finding it in production.

On the client, the cases are simpler: the tile renders only positive counts, and the profile separates "hidden" from "empty" by reading the flag rather than the number.

## The short version

- Decide the privacy rule in one place and make every UI a derivative of it. The badge should not know what "private" means.
- If any surface exposes a set whose size is the number, hide the set too. A list is a count.
- Send the reason, not just the number. A zero with a flag is honest; a bare zero is a claim.
- Keep the bypass role-based so pricing can never change what a user sees.
- Reset account-scoped settings on sign-out, and flush debounced saves on the way out.
- Batch the count for the page, and gate the expensive role check behind an actual need.

The project already had a rule that defaults belong to the user rather than to the product's convenience — we wrote about that in [the post on pre-ticked notification prompts](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-01-notification-prompt-defaults-to-no.html). A friend count visible to everyone is the same kind of default, and once you accept that the user owns it, the list problem follows immediately. The rows that display the result are the same search surface described in [searching every chat and every message](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-18-search-every-chat-and-message.html).

Which is the harder engineering problem in your app: showing a number, or making it disappear without leaving a shadow of it somewhere else?
