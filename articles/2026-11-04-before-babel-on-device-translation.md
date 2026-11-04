---
layout: post
title: "Read chats in the language you study: on-device translation in Flutter"
description: "Folcadi shows incoming messages in the language you're studying — translated in place on the device with ML Kit, offline, one tap for the original."
tags: flutter, languagelearning, mobile, folcadi
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-04-before-babel-on-device-translation.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-04-before-babel-on-device-translation.png
---

Translation is usually a one-way door: the app guesses your mother tongue and everything lands in it. Before Babel opens the other direction. You choose the language you want to read messages in — often the language you are learning — download a language pack, and every incoming message is rendered in it, in the bubble where the text always was.

Nothing else in the app moves. Same chat list, same bubbles, same replies. Only the text changes, converted on the phone, offline, with no server in the loop.

This is how the mode works, why it is built around language packs instead of a translation API, and the three bugs that only appeared once the pipeline ran on real conversations.

## What "in place" means

Most translation features add something: a second bubble, a "translated" section under the message, a spinner, a language label. Every one of those is a reminder that you are reading a translation, and every one of them changes the rhythm of a chat.

Before Babel adds nothing. A bubble contains either the original text or the translated text, in the same place, with the same layout. Someone who did not know the mode existed would not notice a UI change.

Two affordances keep it honest:

- Long-press a translated message to **Show original**; the same action flips it back.
- **Copy** copies the text you are currently looking at, not the text underneath it.

If the text is already in the language you chose, or the message is emoji-only, nothing is translated. The bubble shows what was sent.

## The learner's angle

Language exchange apps are built around your mother tongue. The recommendation is always "read foreign messages in the language you already speak". That is backwards for someone who is studying.

Consider someone learning Korean whose family writes in English. They set the read language to Korean, and the family group becomes daily reading practice in text they actually care about — not textbook sentences about airports, but "did you eat", "call me when you land", "your cousin passed the exam". Tap any message to check the original when the guess is wrong. Reply in English; nobody on the other side sees anything unusual.

That is the whole idea: **immersion using conversations that were going to happen anyway**, with no tutor to schedule and no second app to open.

It is also why the feature does not reuse the profile's mother language. Mother language describes how you want the product to talk to you. The read language describes what you are practising. They are different settings, and conflating them was the first thing we removed.

## Language packs, not a translation service

The translation runs through Google ML Kit's on-device models. Each language pack is around 30 MB, downloaded from the platform's catalog and stored on the device.

The consequences are the reason this design works:

- **The message never leaves the phone.** Language identification and translation are local calls. There is no queue, no API key, no per-message cost, and no text retention to explain in a privacy policy.
- **It works offline.** Once a pack is downloaded, a flight or a metro tunnel changes nothing.
- **Storage is explicit.** The settings page lists every supported language with a Download button, and downloaded packs move to their own section with a switch and a delete button. The user can see exactly what is taking space and remove it.

ML Kit supports 59 languages; the app's own list is around 130. The mode offers the intersection and shows the rest translated not at all — the original text simply stays. On Android it needs Google Play services; where the platform has no on-device translation, the mode reports itself as unavailable instead of pretending.

The server-side translation endpoint that powers the manual **Translate** action is untouched. That action still runs when you ask for it, in your mother language, and still shows its own translated section. Before Babel is a separate display mode with its own state, so the two never fight over the same bubble.

## Inside the pipeline

Every incoming message is queued, identified, and — if a pack for that language exists — translated into the active read language. The result is written to a per-message field and cached under a key that includes both the target language and the message id.

The skip rules are where most of the behaviour lives:

```dart
if (sourceCode == null || sourceCode == target.code || sourceCode == 'und') {
  return;
}

// Translators bake their target in at creation: rebuild them when the
// active language changes, or messages keep arriving in the old language.
if (_translatorTargetCode != target.code) {
  await clearTranslators();
  _translatorTargetCode = target.code;
}
```

Three details matter more than they look:

1. **Own messages are skipped.** Translating your own text into the language you are studying teaches you nothing and doubles the work.
2. **Already-translated messages are skipped.** The cache key is per target language, so switching the read language re-translates rather than showing a stale result.
3. **A source model may be downloaded on first contact.** If someone writes in a language you have never received, ML Kit fetches that source model at translate time. That is deliberate — "translate from anything into my chosen language" is the promise — but it means the first message in a new language can be slow.

The queue processes one message at a time with a short gap, and pauses when the app is backgrounded. A burst of fifty messages should not heat the phone or stutter the scroll.

## The edge cases that shaped the design

Three problems only showed up once the pipeline ran against a real chat, and all three are invisible on a synthetic test with five messages.

### Newest messages first

Opening a chat queued the history oldest-first, which is the order the messages arrived in and the wrong order for the person holding the phone. The screen sits at the bottom of the conversation; translating the top of it means nothing visibly changes.

The fix was to reverse the backfill and bound it to the most recent thirty messages, then give live incoming messages a priority slot at the front of the queue. The message that just arrived is translated before the history behind it.

### Translators are bound to their target

A translator instance is created for a source/target pair. Ours were cached by source language, so switching the read language from Korean to English kept reusing translators built for Korean. The first few bubbles looked right — they were already translated — and then new messages arrived in the previous language.

The fix tracks the target each cached translator was built for and rebuilds the map whenever the user switches. The comment in the snippet above exists because this bug will come back if someone "optimises" that check away.

### Turning the mode off

Disabling the mode stopped the queue but left the translated text on screen: no active target meant no re-render, so bubbles kept showing text the user had just switched away from. Clearing the in-place text, the stored language code, and the show-original flag on disable fixed it — including the case where the user deletes their last pack instead of touching the switch.

## What it costs, and where it stops

- **~30 MB per language model.** Multiple languages mean multiple packs; deletion is one tap.
- **Packs are device-local.** They are not synced across devices and not tied to the account.
- **Android needs Google Play services**; without them the mode says so instead of failing silently.
- **Desktop and web are not supported.** The pipeline degrades to the original text rather than throwing.
- **Quality is a translation engine's quality.** Short, everyday messages are the sweet spot. Idioms, slang, and long technical paragraphs are where it struggles, which is exactly why the show-original toggle is one long-press away.

## Testing the parts that are easy to get wrong

Two widget tests cover the behaviour that regressions would break quietly:

- one asserts that a bubble swaps to the translation in place when the translated text arrives, with no scroll and no rebuild of the list;
- one asserts the queue contract: newest-first backfill, the thirty-message bound, live-message priority, and duplicate skipping.

Neither is clever. Both exist because the bugs above were ordering bugs, and ordering bugs survive green tests that only check that translation happens at all.

## FAQ

**Does my message text leave my device?**
No. Language identification and translation are on-device calls. Nothing in this mode talks to a server.

**Does it replace the manual Translate action?**
No. That action still exists, still uses the server engine, and still renders its own section. The two are independent.

**Can I read a language I have not downloaded?**
No. The pack has to exist on the device; that is what makes the mode work offline.

**What happens in group chats?**
Same pipeline. Every incoming message is identified and translated individually, so a group with four languages is just four source models.

**Can I go back to the original text?**
Per message, yes — long-press and choose **Show original**. Turning the mode off reverts everything at once.

Before Babel is the feature we wanted for ourselves: the app becomes a reading exercise made of your own conversations. If you are studying a language, the most useful text you can read is a message from someone who actually wants to talk to you.
