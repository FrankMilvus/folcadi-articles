---
layout: post
title: Small polish, better feel — a smoother connection status and a clear way out of a call
description: Two small UI changes in GwinG today — the friends connection status now fades in instead of popping, and the call end card gained an explicit Back button.
tags: flutter, showdev, ux, webdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-12-small-polish-connection-status-and-call-end-card.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-12-small-polish-connection-status-and-call-end-card.png
---

Today's changes in **GwinG** were small on purpose. Neither is a headline
feature — but they're the kind of thing you notice the moment it's right: the app
feels calmer and gets out of your way.

## A connection status that fades, not pops

Next to the Friends wordmark we show a live connection label — *Connecting…*,
*Syncing…*, or *No connection*. It used to appear and disappear in a single
frame, which read as a hard blink every time the network changed its mind.

It now **fades and slides** in and out instead of popping, and because it's a
visibility change rather than an entrance, we made it something you control:
there's a new **Connection status** toggle under **Settings → Motion**, joining
the existing motion categories (nine now). When motion is switched off, the label
simply updates instantly — no special-casing needed.

A couple of details we cared about:

- The label only cross-fades when it appears or disappears. Switching between two
  live states updates in place, so you don't get a distracting flicker between
  similar words.
- The wordmark never shifts. The slot is always laid out, so the header stays put
  and the text keeps sitting exactly on the wordmark's baseline.

## A clear way out of a call

When a call ends, GwinG shows a short **end card** — who it was with, how it
ended, how long it lasted, and a **Call again** button. The card returns you to
the app on its own after a few seconds (a little longer when there's a reason to
read), and tapping anywhere dismisses it early.

The catch: "tap anywhere" is not the same as an obvious way out. So we added a
**Back** button right next to *Call again*. Now leaving the end card is a named,
one-tap action instead of a hidden gesture or a short wait.

## Why we bother with the small stuff

A messenger is mostly made of these moments — a label that blinks, a screen you
can't quite leave. Big features get the attention, but the small ones are what
make an app feel considered. We write them up as we ship them, so the polish
gets its share of the credit too.

— the GwinG team
