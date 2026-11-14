---
layout: post
title: "Making an app's map respect dark mode without running a tile server"
description: "One shared theme-aware map in Flutter: a dark basemap from the provider, an attribution widget that cannot go missing, and a bounded tile cache."
tags: showdev, flutter, maps, ux
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-14-theme-aware-maps-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-14-theme-aware-maps-flutter.png
---

A theme-aware map in Flutter sounds like a styling task: pick a tile URL, invert it in dark mode, move on. It is really an ownership task. Four maps in our app — a search result preview, the fullscreen search map, a profile's city card and the fullscreen city map — each carried their own copy of one hardcoded light tile URL. Every other surface went dark correctly, and those four stayed bright, which made the app look broken exactly where it was trying to show something useful.

There is now one tile layer: it resolves a style from the app's brightness or the user's choice, asks the basemap provider for that style, carries the credit the map licences require, and sends every request through one shared tile cache.

## Four maps, four copies of the same URL

The duplication was invisible because the copies were identical. Nothing drifted, so nothing looked wrong until the app got a dark theme and four rectangles did not join in.

Identical copies are still a maintenance problem, because the cost arrives all at once. Adding a dark style, a cache or attribution is one change per map — and the map that gets skipped is always the one nobody remembers, usually the small preview inside a profile card. A shared widget turns that from a discipline problem into a default.

The four maps differ in ways that matter: three are interactive and one is a card that accepts no gestures at all, two sit inside a rounded clip on a page and two are fullscreen routes with a floating back button. One shared layer has to work in all four contexts without four sets of options.

There was a smaller symptom: two maps inherited the library's default light grey background, so opening them in dark mode flashed pale grey until the first tiles painted. That colour is now one helper, `mapBackdropColor`, returning the theme's surface colour.

## A theme-aware map in Flutter needs one owner

Every map now adds the same two widgets: `MapTiles` for the basemap and `MapAttribution` last, with markers and labels between them.

The style decision is one pure function. A user choice of Auto, Light or Dark wins; Auto defers to the app's brightness:

```dart
MapStylePreference resolveMapStyle(BuildContext context) {
  return switch (mapStylePreferenceNotifier.value) {
    MapStylePreference.light => MapStylePreference.light,
    MapStylePreference.dark => MapStylePreference.dark,
    MapStylePreference.followTheme =>
      Theme.of(context).brightness == Brightness.dark
          ? MapStylePreference.dark
          : MapStylePreference.light,
  };
}
```

The overrides earn their place: some people read a dark app with a light map because a dark basemap hides the roads they want, and some want the reverse. The preference is stored locally and kept with the account's profile settings, so it travels between devices.

The tile layer sits inside a value listener on that preference, so switching styles re-requests tiles immediately — no restart, and no need for the parent screen to rebuild.

The tile provider is created once per map instance, because the layer takes ownership of the provider it was built with. It cannot be hoisted to a global; a provider a disposed map has torn down will break the next one. Shared behaviour belongs in a proxy, not a shared instance.

## The dark style has to come from the provider

The primary basemap source publishes a light and a dark raster style from the same tile path, high-density variants included. With it configured, dark mode gets a designed dark basemap: dark land, light labels, roads that read as roads.

The fallback is the public OpenStreetMap tile server, which is light-only, and it exists for two different reasons.

The first is a build without a basemap identifier. Development builds and anyone compiling from source get the open tiles exactly as before; nothing is required to make a map appear.

The second is a failing request while the identifier is present. The tile layer accepts a per-tile fallback URL, so a request that errors serves an open tile in that spot instead of leaving a hole. A briefly mixed map is ugly; an empty grid is worse.

Between those cases sits the honest stopgap. Without the provider's dark style, dark mode would be a dead setting, so the layer applies a per-tile colour filter — invert plus a 180° hue rotation — when, and only when, the style resolves to dark and the open tiles are in use. Labels survive the inversion and stay readable, but the hierarchy flips: roads end up darker than the land they cross, so the map reads as a photographic negative rather than a night map.

That is fine at 180 dp, where the map anchors a city name and opens a fullscreen view. It would not be fine as the fullscreen look, which is why the dark style exists and why the code calls this a stopgap.

The filter is applied at paint time, not download time, so cached bytes stay unfiltered and flipping between light and dark re-renders from the same cache. Once the provider's dark style is in use, no filter is applied — including to the open tile that may appear after a failed request.

## Attribution is a licence term, not polish

Both tile sources require visible credit. That is not a nicety for an about screen, and not a comment in the source: OpenStreetMap's tile policy asks for the credit to stay visible, and the basemap provider's terms say the same.

So the credit is a widget, `MapAttribution`, and it is the last child of every map. It draws a single line of small text — around nine logical pixels, on the theme's surface at a little under full opacity — with a semantics label, and a tap opens the copyright page. Because it lives in the shared layer, a fifth map inherits the wording and the placement by copying two lines, not by remembering a licence.

Layout is where the copies used to diverge. The default corner is bottom-right, the convention. Two maps cannot use it: the profile city card has a location chip in one bottom corner and an avatar over the other, so its credit is anchored top-left; both fullscreen maps float a back button in the bottom-right, so they pass padding that lifts the label clear. The widget takes an alignment and an inset instead of a boolean, and the awkward cases are handled at the call site without a second copy of the label.

Two details help it survive real layouts. It is one line, so it never wraps on a narrow card, and it sits inside the preview's rounded clip, so a corner radius cannot shave off half the credit. The wording follows the sources in use: both names when the provider is configured, only the open source when it is not. Crediting a provider that sent no tiles is its own kind of wrong.

## A decoration still costs requests

The profile city card is 180 dp tall, accepts no gestures of its own, and opens a fullscreen map when tapped. It is decoration with a door. It also pulls somewhere around forty to sixty tiles when it renders, and a fullscreen map pulls a similar number — and without a disk cache all of that came back after every restart.

That is because the library used to keep only an in-memory image cache. Kill the process and the tiles are gone. The version we moved to added a built-in on-disk store that honours HTTP cache headers, which turns "the same profile opened twice" from a network event into a file read.

Caching matters for two less obvious reasons. The basemap provider counts tile requests against a monthly free allowance, so every cached tile is allowance left for someone exploring the map. And OpenStreetMap's policy asks clients to cache rather than repeatedly hit a donated server.

## A size cap and a switch the user can find

`MapTileCache` owns the cache for all four maps. The limit is a setting in Storage and Downloads: off, or 50, 100, 250, 500, 1000, 2000 MB, defaulting to 200 MB. It is persisted locally and kept with the account settings, so a chosen cap survives a reinstall or a device change.

The library's cache configuration is immutable once the instance exists, which forces an awkward lifecycle: changing the limit means destroying the cache and creating a new one. That raises two real problems.

The first is ordering. The limit can change from the settings sheet and from a profile-settings sync in the same session, so reconfigurations are queued and a superseded one is skipped. The last requested limit wins, and an older destroy cannot land after a newer create.

The second is reach. A map already on screen holds the provider it was built with, so replacing the cache instance would never reach it. Instead every map receives the same stable proxy provider, which resolves the active cache per request, so changing the cap affects everything without rebuilding a map.

Clearing is the same operation with a delete: files are removed and the cache is re-created with the current limit, so maps keep caching instead of quietly falling back to the network forever.

One policy runs through the class: every read and write failure is swallowed as a cache miss. A torn-down cache, a corrupt entry, a permission problem — each logs a line and lets the network win. A cache that can blank a map is worse than no cache, because the failure looks like the feature.

## What the cache key must not contain

Keys are built from the tile URL without its query string:

```dart
static String _tileKeyGenerator(String url) {
  final uri = Uri.tryParse(url);
  final stable = uri == null ? url : '${uri.scheme}://${uri.host}${uri.path}';
  return BuiltInMapCachingProvider.uuidTileKeyGenerator(stable);
}
```

A tile's identity is its style, zoom, column and row. Credentials are not part of it: keying on them means a rotated client identifier orphans the whole cache, so you pay the full download cost exactly when you changed a configuration value. Keeping the query string out also means a cache entry cannot leak a credential onto disk.

## What this pattern does not solve

The dark style is only designed when the provider is configured; everything else is the inverted stopgap, which the product should say out loud.

Caching helps repeated views, not the first paint. The first open of a fullscreen map still fetches tiles, and the backdrop colour is what stops that from reading as a failure.

Marker avatars were out of scope. The style resolution and the "should this tile set be darkened" predicate are unit-tested because they are pure functions; the rest is visual and needs a device pass in light, dark and an AMOLED theme, plus a deliberately failing provider.

## The short version

- Resolve the map style once, from an explicit preference that falls back to the app's brightness.
- Prefer a basemap style published by the provider; treat an inverted light map as a stopgap, and say so.
- Fall back per request, not per map: a failing tile should degrade to another source, not to a hole.
- Make attribution a widget in the shared layer. Reposition it, never remove it, and let its wording follow the sources in use.
- Bound the cache, expose the limit, and make clearing it a real operation.
- Key cached tiles by what the tile is, never by what the request carried.
- Set the backdrop to the theme's surface so the first frame is not a grey flash.

The same lesson showed up in our [floating glass header work](https://frankmilvus.github.io/folcadi-articles/articles/2026-11-13-floating-glass-chrome-flutter.html): once one widget owns the appearance, the screens stop disagreeing. A map is just a surface that happens to come from somewhere else.

If you have shipped a map inside a dark app, I would like to know which corner your attribution lives in — the honest answer for us was "not always bottom-right".
