---
layout: post
title: "Flutter reconnect catch-up: the chat that stayed blank"
description: "A notification tap on a weak link opened an empty chat that never filled in. The fix was client-side Flutter reconnect catch-up plus a bounded retry."
tags: showdev, flutter, websocket, reliability
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-21-chat-reconnect-catchup-flutter.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-21-chat-reconnect-catchup-flutter.png
---

You tap a message notification on a train, in a lift, in a basement café. The chat opens. The message is not there. The connection comes back, and the message is still not there. You leave the chat and open it again, and the message appears instantly.

That last sentence is the bug report, and it is also the diagnosis. If re-entering fixes it, then entering was the only thing that ever fetched anything. Reopening a chat triggered a path that the app had no other way to reach. The work behind this fix was not about making the connection better — it was about giving the open chat a reason to ask for its history again. Flutter reconnect catch-up, in one sentence: when the socket says it is back, refetch.

## Two client-side gaps, one symptom

Neither of the two gaps looks like a bug in isolation.

The first was that history was fetched once. When a chat opened, the controller hydrated from the local cache and then asked the server for the newest page. If that request failed — a timeout on a weak link is the ordinary case — the code fell back to the cache and stopped. That fallback is correct: showing a stale conversation beats showing an empty screen. What was missing was a second attempt. A failed fetch was treated as a final answer.

The second was subtler. A socket that was already connected would never emit a reconnect event. So if the history request failed while the connection itself stayed up, nothing in the app was scheduled to try again: no reconnect, no retry, no second fetch. The chat sat on an empty cache with a healthy-looking connection indicator, which is the worst combination to debug from a user report.

Add them together and you get the report exactly. The notification tap opens the chat during the weak spell; the fetch fails; the socket recovers without ever "reconnecting"; the failed fetch stays invisible.

## "Connected" is not "in sync"

Most realtime clients are built on an implicit assumption that the connection state is a proxy for the data state. It is a good assumption right up until the moment it is not.

A socket being up tells you that future events can arrive. It says nothing about events that already happened while it was down. Our server does not replay what a client missed, so anything that occurred during the gap is simply absent from the client's view until something fetches it. The reconnect event is a hint that a gap may exist, not proof that it has been closed.

That is why the fix belongs on the client. Server-side replay buffers are a fine design, but they change the contract for every client and every event type. A catch-up fetch is local, it uses the endpoint that already exists, and it produces the right result whether the gap was two seconds or two hours.

## Flutter reconnect catch-up: refetch when the socket returns

The controller already listened for connection state changes; it re-sent the signal that tells the server which chat is currently on screen. The change was to do the same thing a user does by re-entering the chat: reload the history.

```dart
void _onConnectionStateChanged() {
  if (_disposed) return;
  notifyListeners();
  if (ws.connectionState.value != WsConnectionState.connected) return;
  if (!_isAppInForeground) return;   // see below
  _sendActiveChat();
  unawaited(loadMessages());
}
```

Two guards sit in front of the fetch, and both matter.

The first is `_disposed`. A controller that has been torn down must not start work, and a reconnect can arrive at an awkward moment.

The second is the foreground check. Registering a chat as open is how the server knows not to send a notification for messages the user is already looking at. Doing that from the background would suppress notifications for a chat nobody is reading — the opposite of the intent. So the catch-up only runs while the chat is actually in front of someone. When the app comes back to the foreground with this chat still on top, the same loader runs again, and that is the case where catch-up matters most.

The value of routing this through the existing load path is that there is only one loader. Catch-up, first open, a return to the foreground and a scheduled retry all end up in the same function, so a fix to one is a fix to all of them, and a bug in one is at least reproducible in the others.

## A retry ladder that knows when to stop

The reconnect hook fixes the case where the socket drops and comes back. It does not fix the case that started this: a socket that never drops while the fetch fails.

So a failed history fetch now schedules a retry: two seconds, then four, eight, fifteen, thirty, and finally sixty. The list is deliberately short. Six attempts, capped at a minute apart, covers a tunnel or a lift ride without turning a broken network into a background poller. After the last delay, the retry stops and the chat stays on its cached content; the next reconnect, resume or manual entry starts the cycle fresh.

```dart
static const List<int> _historyRetryDelaysSeconds = [2, 4, 8, 15, 30, 60];
```

Three reset rules keep the ladder honest:

- **Success resets the counter.** A chat that recovers after two attempts and later fails again starts at two seconds, not at sixty.
- **Background cancels the timer.** Retrying from the background burns battery on a request the user cannot see, and on some platforms it will not run anyway.
- **A new attempt cancels the pending timer.** Any load — reconnect, resume, the user re-entering — makes the scheduled retry redundant, and two overlapping ladders would drift apart.

`dispose()` cancels it too. A timer that outlives the chat is how you get a request for a conversation that no longer exists on screen, and a controller retained by a closure it should not have.

This retry ladder is separate from the socket's own reconnection backoff. They are two different questions: how fast should the transport re-establish itself, and how fast should the data layer ask again? Coupling them would make one schedule serve two purposes badly.

## One fetch at a time, three ways in

At this point there are three triggers that can ask for history: a reconnect, a return to the foreground, and a scheduled retry. A user re-entering the chat makes four. Without coordination, a flaky link produces exactly the burst you would expect — several concurrent requests for the same page, each one racing the others into the message list.

The controller already had an in-flight guard, and it does the work here. If a load is running, a second call does not start another fetch. It records that a follow-up refresh was requested and returns the running operation's future, so every caller waits for the same result instead of racing.

The follow-up flag matters as much as the guard. "Ignore this call" would drop a legitimate refresh that arrived after the in-flight request had already read its data. "Queue one more" keeps the newest trigger meaningful without opening a second connection to the server.

```dart
if (_loadingMessages) {
  _reloadMessagesRequested = true;
  return _loadMessagesCompleter?.future ?? Future<void>.value();
}
```

A detail worth keeping: callers await that future. A first load that returns early while the list is still empty makes the view flash its "no messages yet" state before the data lands, which reads as a different bug entirely.

## What the tests pin down

Two regressions cover the report directly, both in the initial-load suite:

- An offline first load followed by a socket reconnect delivers the history without the user reopening the chat.
- A failed fetch while the socket stays connected is recovered by the bounded retry.

The second test needs the backoff curve to be compressible; waiting real seconds in a widget test is not an option, so the delay function has a test seam that maps the attempts to milliseconds. That seam is the difference between a retry that is tested and a retry that is merely written.

The first test is the one I care about most, because it encodes the lesson rather than the mechanism: the chat must recover from a reconnect on its own, without the user performing the workaround.

## What this does not fix

The catch-up fetch assumes the server can answer it. If the server's own view of the conversation is incomplete — events that never reached it, state that is simply behind — refetching will return the same incomplete truth, and no amount of client retrying changes that. Keeping the two cases apart matters: our fix makes the client stop being the reason a message is missing. It does not make the backend perfect.

It also does not make the local cache smarter. A cache that is behind is still held back instead of being shown as current, and a cache that is ahead simply reflects what arrived while the app was open. The reconnect fetch is what reconciles both.

## Rules I would keep for any realtime client

- **Treat "connected" as permission to fetch, not evidence of freshness.** The transport knows about the future, not the past.
- **Give the open screen a catch-up path.** Reconnect, resume and retry should all funnel into the same loader, so there is one behaviour to reason about.
- **Bound the retries.** A short ladder with a hard stop looks after the battery and still survives a tunnel.
- **Reset on success, cancel on background, cancel on dispose.** Three lines that prevent a retry from becoming a poller.
- **Coalesce the triggers.** An in-flight guard plus a queued follow-up is enough; concurrent fetches for the same page are never worth it.
- **Test the recovery, not the timer.** The assertion that matters is that the message appears after a reconnect, not that a `Timer` was constructed.

We wrote about a similar instinct in [the post on calls that never connected](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-29-flutter-webrtc-calls-that-never-connected.html): in a realtime feature, the interesting failures are almost never in the happy path, they are in the recovery path that was never exercised.

The user-visible outcome is small and worth stating plainly: a chat opened from a notification on a bad connection now fills itself in when the connection recovers. No re-entry, no force close, no waiting for the next message to arrive and pull the history with it.

What is the last bug in your app that a user could "fix" by closing and reopening the screen? That workaround is usually a missing recovery path wearing a disguise.
