---
layout: post
title: "Automatic voice-message transcription in Flutter: a quiet queue"
description: "How Folcadi transcribes incoming voice messages automatically in Flutter: a newest-first queue, a cache-first fallback chain, and failures that stay quiet."
tags: flutter, dart, mobile, showdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-10-automatic-voice-message-transcription-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-10-automatic-voice-message-transcription-flutter.png
---

A voice message arrives. You tap play, and the text is already underneath the
waveform. It feels like the app read your mind. Under the hood it is a small
queue with a few strict rules: transcribe what is new, never backfill history,
do one thing at a time, and fail without making noise.

That last rule is the one users notice most. Automatic transcription is a
background convenience, not a modal task. If the remote service is down, the
voice note simply stays untranscribed; the user can still swipe to try again.
No snackbar storm, no red banners, no “something went wrong” every few seconds.

This post walks through the design of the automatic queue in Folcadi, a
language-exchange messenger built with Flutter: why it only handles live
messages, how the fallback chain works, why the “transcribe my own voice
messages too” switch is separate, and how the on-device model downloads survive
navigation.

## Automatic should mean “from now on”, not “all history”

The first product decision is the scope. When automatic transcription is on,
every voice message that arrives **from now on** is transcribed. Past messages
are never backfilled.

That is not a technical limitation; it is a deliberate choice:

- **Predictability.** Turning on a switch should not start a long, invisible
  job over years of chat history.
- **Battery and data.** Decoding audio is expensive. A backfill can burn
  battery and mobile data for messages the user may never open again.
- **Privacy.** Old voice notes are not automatically turned into text just
  because a setting changed later.
- **Storage.** Every transcription is cached. Backfilling a large history
  creates a cache the user did not ask for.

Live-only also makes the feature easy to explain: “voice messages that arrive
while this is on will transcribe themselves.” If the user wants an older note
transcribed, the manual action is still there.

The default is **on**. The separate switch for the user’s own recordings is
**off**. That asymmetry is intentional: incoming messages are the common case,
while transcribing your own voice is a different kind of consent.

## One queue, one transcription at a time

The automatic path is a singleton queue, not a set of independent futures. A
burst of voice messages should not start five decoders at once and fight for
the CPU. The queue processes **newest first**, strictly sequentially, with a
small gap between items so the UI stays responsive.

A simplified version of the core looks like this:

```dart
class AutoTranscriptionQueue {
  final Queue<MessageVM> _queue = Queue();
  bool _processing = false;
  bool _foreground = true;

  void enqueue(MessageVM message) {
    if (!_enabled) return;
    if (!_eligible(message)) return;
    if (_queue.any((m) => m.message.id == message.message.id)) return;

    _queue.addFirst(message); // newest first
    _drain();
  }

  Future<void> _drain() async {
    if (_processing || !_foreground || !_enabled) return;
    _processing = true;
    try {
      while (_queue.isNotEmpty && _foreground && _enabled) {
        final message = _queue.removeFirst();
        await _process(message);
        await Future<void>.delayed(const Duration(milliseconds: 80));
      }
    } finally {
      _processing = false;
    }
  }
}
```

The important parts are not the names; they are the gates.

**Newest first** means a fresh message does not sit behind a backlog. The queue
is not a FIFO archive; it is a small work list where recency wins.

**Deduplication by message id** matters because the same message can be
enqueued from more than one path. The websocket handler enqueues incoming
messages; the settings toggle can kick the queue; the server echo for an own
message is another entry point. Without a dedup check, the same voice note can
be transcribed twice and the second result can race the first.

**A re-check before work starts** is just as important. A message may have been
transcribed manually while it sat in the queue. The worker checks the current
state again before making a request, so the manual and automatic paths do not
fight over the same bubble.

**Foreground gating** pauses the queue when the app goes to the background and
resumes it on return. A background transcription is not worth the battery
surprise, and mobile platforms are increasingly strict about background work.
The queue keeps its place; it just stops draining.

## The fallback chain: cache, remote, on-device

Every transcription path in the app shares the same chain:

1. **Cache first.** If the message already has a cached transcription, use it.
   The server and offline results are cached separately so the UI can keep the
   right label across restarts.
2. **Remote service.** If there is no cached result, send the audio to the
   transcription service.
3. **On-device fallback.** If the remote call fails and the user has enabled
   offline transcription with a model installed, run the audio through an
   on-device Whisper model.

The cache is not an optimisation detail; it is part of the UX. A transcription
that survives a restart should not disappear and reappear. The offline and
remote results are stored under different keys because they are different
claims: “the remote service transcribed this” and “this device transcribed
this”. The label in the chat bubble reflects which one produced the text.

The fallback chain also explains why the automatic path can stay quiet. The
manual action has a snackbar UX because the user explicitly asked for a result
and deserves feedback. The automatic path has no user waiting on a tap. If all
three steps fail, the correct outcome is silence: the message remains
untranscribed and the manual action is still available. Logging the failure is
enough for debugging; a toast is not.

That distinction is easy to miss. A shared service that always shows an error
is convenient to write and terrible to live with. The automatic path and the
manual path can share the same transcription logic while keeping different
failure policies.

## Why own voice notes are a separate switch

Transcribing incoming messages is mostly about comprehension: you can read a
voice note in a noisy place, search it later, or translate it. Transcribing
your own recordings is a different feature. It can be useful, but it is not the
same default.

The own-message switch is therefore separate and off by default. When it is on,
the queue does not enqueue the optimistic bubble that appears the moment the
user stops recording. It waits for the server echo that replaces the temporary
message with the real one. At that point the audio upload has finished and the
message has a stable id, so the transcription request has something real to
work with.

That detail prevents a class of subtle bugs:

- The temporary bubble has no durable message id, so caching would be wrong.
- The audio may still be uploading, so a request would fail or duplicate work.
- The echo also confirms the message was accepted, which is a better moment to
  start background work.

In other words, the queue is not just “transcribe every voice message”. It is
“transcribe every eligible voice message at the moment it becomes a real,
addressable message”.

## Encrypted chats stay out of the queue

Encrypted chats are deliberately excluded from automatic transcription. The
audio is ciphertext at rest and in transit through the server, so there is no
plaintext for the service to transcribe. The client also does not enqueue those
messages in the first place: the eligibility check rejects encrypted content
before any request is made.

This is a defence-in-depth choice. The server should refuse the request anyway,
but not sending it is cheaper, clearer, and avoids a pointless round trip. The
UI hides transcription in encrypted chats for the same reason: a feature that
cannot work should not look like it might.

The broader lesson is that a background convenience feature needs an explicit
eligibility boundary. “Is this a voice message?” is not enough. The queue also
asks: is it encrypted, is it already transcribed, is it in flight, and is it
eligible under the current settings?

## Downloads that survive navigation

The on-device fallback only works if the user has a model. The model manager
offers three sizes — roughly 74 MB, 141 MB, and 465 MB — so the user can choose
the trade-off between speed, accuracy, and storage.

The download path has a few rules that matter more than the model picker UI:

- **Stream to disk, never buffer in memory.** A 465 MB model should not become
  a 465 MB list in RAM. The download writes chunks to a `.part` file and
  renames it only after the size sanity check passes.
- **Keep the busy state in the service, not the page.** If the user navigates
  away mid-download, the progress should still be there when they return. The
  page is a view; the download is a service-owned job.
- **Ignore duplicate taps.** A second tap on the same model should join the
  in-flight download, not start a second writer for the same file.
- **Publish progress in whole percents.** The HTTP stream yields thousands of
  chunks for a 74 MB file. Rebuilding the UI for every chunk is wasted work.
- **Fire a completion notification.** The user may have left the settings page
  long ago. A local notification closes the loop without forcing them to keep
  the app open.
- **Use a foreground service where the platform requires it.** The download
  should survive the app being minimised, and the user should know it is
  happening.

The sanity check is worth calling out. A truncated model file can look like a
successful download to a naive client and then crash the native engine when it
is loaded. Checking the expected size before the rename turns a mysterious
crash into a clear retry.

## Settings that follow the account

The toggles are stored locally for instant startup and synced to the account
profile so they survive a reinstall. That includes the automatic switch, the
own-message switch, the offline switch, the installed model list, and the
active model.

The sync is not just convenience. A user who enabled offline transcription,
downloaded a model, and then reinstalled the app should not have to rediscover
the whole setup. On the next login, the app can restore the preferences and
re-download the missing assets in the background.

There is a subtle rule here too: the local value should win for the first
frame, and the server value should reconcile afterwards. Blocking startup on a
network round trip makes the settings page feel broken on a weak connection.
The same principle as the transcription queue applies: do the visible thing
first, then reconcile in the background.

## The rules I would keep

Automatic voice-message transcription looks like a small feature. Most of the
work is in the boundaries:

- **Default on, but no backfill.** The switch changes future behaviour, not
  history.
- **One queue, newest first.** A burst should not turn into a decoder race.
- **Cache before network.** A result that survives a restart is part of the
  feature, not an optimisation.
- **Quiet failure in automatic mode.** The user did not ask for this specific
  transcription; do not interrupt them when it fails.
- **Loud, actionable failure in manual mode.** The user did ask; give them a
  retry or a setup path.
- **Separate consent for own audio.** Incoming messages and your own recordings
  are not the same default.
- **Exclude encrypted content early.** Do not send what cannot be transcribed.
- **Model downloads belong to the service.** Navigation should not cancel a
  download or lose its progress.

None of these rules is specific to transcription. They apply to translation,
image processing, prefetching, and every other background convenience a chat
app accumulates. The feature is the queue; the product is the discipline around
it.

## FAQ

### Does automatic transcription process old voice messages?

No. It only handles voice messages that arrive while the setting is on. Past
messages are never backfilled. You can still transcribe an older message
manually.

### Does it work without a network connection?

Only if offline transcription is enabled and a model is installed. Otherwise
the remote path fails and the message stays untranscribed until you retry.

### Are my own voice messages transcribed automatically?

Not by default. The separate “Transcribe my voice messages too” switch is off.
When it is on, the queue waits for the upload to finish before it starts.

### Why are encrypted chats excluded?

The audio is ciphertext, so there is no plaintext for the transcription service
to read. The client does not enqueue encrypted messages, and the feature is
hidden there rather than pretending to work.

### What happens when the transcription service is down?

In automatic mode, nothing visible. The message stays untranscribed and can be
retried manually. In manual mode, the app offers a retry or a path to set up
offline transcription.
