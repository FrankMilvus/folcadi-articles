---
layout: post
title: "Flutter WebRTC calls that never connected: three bugs, three fixes"
description: "Our Flutter WebRTC calls closed right after the offer. Three separate bugs hid behind one symptom: signalling, a relay behind NAT, and background ringing."
tags: flutter, webrtc, showdev, webdev
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-29-flutter-webrtc-calls-that-never-connected.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-29-flutter-webrtc-calls-that-never-connected.png
---

For two weeks, every 1:1 audio and video call in our Flutter app failed the same boring way: the caller saw "Calling…", the screen flipped to "Ended" about a second later, and the callee's phone never rang. The client logs looked fine. The peer connection reached `have-local-offer` and then closed.

The temptation is to pick one suspect and start there. We spent the first evening on the relay server, which was genuinely broken — and fixing it changed nothing, because two other bugs sat in front of it. This is the story of all three, plus the small traps that were still waiting after them.

## The symptom: `have-local-offer`, then silence

Worth stating precisely, because it narrows the search:

- The app created an offer, set it as its local description, and sent it.
- The backend accepted the WebSocket message without an error.
- No `answer` ever arrived. After a timeout, the client tore the call down.

That rules out most of WebRTC theory — codecs, SDP munging, permissions — and leaves one question: did the offer actually reach the other side?

It did not.

## Bug 1: the signalling key pointed at a row nobody created

The app signals calls over the WebSocket. Each event carries a `call_id`, and the backend looked up a `CallSession` by that id before relaying.

The problem is what `call_id` contains: the **conversation** id. The client never creates a call session at all — the table exists, the endpoints exist, and nothing calls them. So the lookup returned `None` for every offer, answer and ICE candidate, and the handler returned early:

```
Call 42 not found
```

Every event was dropped. The call closed immediately after `have-local-offer`, which is exactly what the symptom looked like.

**The fix** was to stop looking for a session and route by membership instead: resolve the participants of the conversation, reject a sender who is not a member, and relay the event to everyone else. The wire format did not change. `call_id` still means the conversation id; the backend re-prefixes the event type (`offer` → `call_offer`) and relays any type verbatim, including ones it has never heard of.

Two side benefits fell out of the change:

- A new signalling event no longer needs a backend deploy to work.
- The dead `CallSession`/`CallParticipant` models became obvious — including an endpoint that used a model it never imported, which only surfaced as a `NameError`. A test now pins that path.

**The lesson:** when a lookup fails on an id, check whether both sides agree on *what the id identifies*. `call_id` was a naming lie, and it cost a week.

## Bug 2: a relay only the server could reach

With routing fixed, calls still failed — but now only sometimes. Office Wi‑Fi worked. Mobile networks did not. That is the classic signature of symmetric NAT: direct peer-to-peer connectivity fails, and everything has to go through a relay.

We had a relay. It was simply unreachable.

The TURN server ran on a machine behind NAT, published through a port-forwarding tunnel. Two things were wrong with that, and only one is obvious:

1. The relay advertised its own **container address** as the relay candidate, so clients were handed an address that does not exist outside that host.
2. Even with the address fixed, a single forwarded port cannot carry TURN. A relay is not one connection: each allocation gets its own UDP port for the peer. The tunnel maps one port; the relay needs a range. And STUN through that tunnel reports the tunnel as the client, so the whole candidate dance is meaningless.

**The fix** was to move the relay to a host that owns a public IP and configure it for the outside world. The settings that mattered, with the real values redacted:

```ini
listening-port=3478
external-ip=<public-ip>         # without this: a private address is advertised
min-port=<first-relay-port>
max-port=<last-relay-port>      # a dedicated UDP relay range, firewall-open
denied-peer-ip=<private-range>  # one line per range: loopback, link-local, RFC 1918
no-multicast-peers
```

Three operational details that are easy to miss:

- **The DNS record must be DNS-only.** If the relay hostname sits behind an HTTP proxy or CDN, the proxy answers the UDP/TCP traffic and the relay never sees it.
- **Open the whole relay range in the host firewall**, not just the listening port.
- **Keep the kernel's ephemeral port range away from the relay range.** Otherwise the kernel can already hold a port the relay wants to bind, and allocations fail intermittently — the worst kind of failure.

The relay is also restricted so it cannot be used as a door into the host's private networks (the `denied-peer-ip` lines above). A relay reachable from the internet is a feature; a relay that will happily forward to the host's own loopback sockets is a vulnerability.

**The lesson:** "peer-to-peer works on my network" is not a test. Try a mobile network, and treat a missing relay as an untested code path.

## Bug 3: the app stops listening when it goes to the background

Now calls connected — while both apps were open. Minimise one, lock the phone, and the call again reached nobody.

This one is architectural, not a typo. The app closes its WebSocket when it goes to the background, deliberately: an idle socket on a sleeping phone is a battery and reconnection problem. But a call offer is a one-shot event. Sending it to a socket that is closed means the offer is simply gone.

**The fix:** the backend now sends a **data-only push** whenever it relays an offer, and the app's background isolate shows a **full-screen call notification** with a looping ringtone, a vibration pattern, and Accept / Ignore / Decline actions. Accept launches the app and auto-answers; the pending call is parked in shared preferences so the UI isolate can adopt it on cold start or resume. There is no foreground service and no native audio code: the notification *is* the ring.

Three details turned out to matter more than the notification itself.

**The push carries no SDP — and it should not.** By the time the callee wakes up, the caller's ICE candidates have already been sent over a socket that was closed. The media path is broken whether or not the callee has the offer. So instead of squeezing an offer through a push, the callee sends an `offer_request` on accept, and the caller builds a fresh offer with `restartIce()`. A bare re-offer would reuse the already-converged ICE credentials and produce **no new candidates**, which is a confusing way to fail. The request also restarts the caller's ring timeout, so a slow wake-up does not look like a hang-up.

**Decline is headless.** The decline action runs in the background isolate, which has no WebSocket. It refuses the call over HTTP, and the server reuses the same relay path it would have used for a socket event. It also echoes a reject back to the declining device, so a ring that is still visible in the running app clears too. Ignore is also headless but signals nothing — it silences this device and lets the caller ring out.

**The push must never block signalling.** Pushes run on the caller's own WebSocket loop, right after the offer is relayed, and the ICE candidates are queued behind them. So the push is time-boxed to 1.5 seconds: an unreachable push service may delay the ring, but it must not stop the call from connecting.

Two Android details, since the ring lives in a notification channel:

- Channel properties (sound, vibration, importance) are **frozen at creation**. Changing the ringtone therefore needs a new channel id, so the app selects one of a small family of channels based on the user's sound/vibration preference.
- A channel's vibration pattern plays **once**, and `FLAG_INSISTENT` reliably repeats the *sound*, not the vibration. The pattern has to span the whole ring.

And one lock-screen rule: waking the screen and drawing above the keyguard is held **only while a call exists**, at runtime. Declaring it statically on the activity looks simpler and is worse — every launch then surfaces over the lock screen, which is how users end up staring at their PIN pad after a call.

## Three smaller traps we hit afterwards

1. **ICE candidates that arrive before the remote description.** Candidates can only be applied once the peer connection has a remote description (the offer on the callee, the answer on the caller). Both sides routinely send candidates before they have it. Applying one too early is rejected by libwebrtc and silently lost — the call "connects", but media never flows. We buffer inbound candidates until the description exists, then flush.
2. **Credential-less ICE entries and JSON `null`.** If a STUN entry is serialised with `"username": null, "credential": null`, some platforms treat the server as *credentialed* and hand it to the wrong initialiser. The fix is to omit the keys entirely. This stays invisible until you try a platform you did not test on.
3. **Devices that cannot capture video.** One of our test devices exposes a camera that the WebRTC library fails to open with a **fatal native error** — not an exception Dart can catch, so the process dumps traces and it reads as a crash. Enumeration does not help: the camera is listed. We now combine identity-based detection, an explicit user override (Always on / Automatic / Always off), and a bounded fallback that degrades a call to audio instead of aborting it.

## Why we were debugging blind

None of the above was findable from the app alone, because of a one-line mistake: the application root logger had **no handler**. Every `logger.info` in the backend — including the "call not found" line that would have ended the investigation in minutes — was silently discarded. The framework was happy to run without one.

Attaching a stdout handler immediately produced the diagnosis. Now the call handler logs one line per relayed event, with ICE candidates at DEBUG so they cannot bury the offer/answer/end lines:

```
call_offer conversation=42 from=1 to=[2]
```

**The lesson:** before adding logging to a subsystem, check that the logging you already have is not being thrown away.

## What it looks like when it works

Audio and video calls now connect end to end: peer-to-peer when the network allows, relayed when it does not, and ringing on a locked phone. If ICE drops mid-call — a Wi‑Fi to cellular handover, for example — the caller renegotiates with a fresh offer, and only the caller ever offers, so the two sides cannot both renegotiate at once. A watchdog rebuilds a first attempt that sits in `connecting` too long, which is the difference between "video shows up in two seconds" and "the first call takes forever". We added [the call-end card](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-12-small-polish-connection-status-and-call-end-card.html) earlier in the same series of polish work, and it finally has something to show.

## What is still missing

Being honest about the edges is part of the design:

- **Group calls.** The relay already fans out to every conversation member, but the client is 1:1 and real group calling needs a different media server.
- **iOS background ringing.** The push is still a background push there.
- **Ephemeral relay credentials.** The credential is static today; short-lived credentials are the next improvement.
- **A real audio-route switch for speaker.** The toggle is cosmetic on some devices.
- **Multi-device calls.** Every device should ring and the first answer should win. That is the next post.

## FAQ

### Why does a re-offer need `restartIce()`?

Because a new offer alone does not re-gather candidates. The ICE credentials are already converged, so the offer reuses them and produces no new candidates — while the callee still has none of the old ones. `restartIce()` forces a fresh gathering round. See the [WebRTC session lifetime notes](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Session_lifetime) and [RFC 8656](https://datatracker.ietf.org/doc/html/rfc8656) for the relay side.

### Why serve the ICE configuration from the backend instead of embedding it?

A credential compiled into the app cannot be rotated, and it can drift from what the relay actually accepts. Serving it means the server is the single source of truth, and the app caches it for the returned TTL. Configuration trouble then degrades a call to peer-to-peer instead of breaking it.

### Why close the WebSocket in the background instead of keeping it open?

An idle socket on a sleeping phone is a battery and reconnection liability, and it does not reliably survive Android's background restrictions anyway. A push for the one event that must not be missed is a much smaller contract.

### Do these fixes apply to group calls?

Partly. The signalling is already conversation-based, so relaying to N participants works. What does not work is the media topology: N participants using N peer connections does not scale, and the client is written for 1:1. That needs a different media server and a different post.

### How do you make the full-screen ring appear on a locked phone?

A full-screen intent on a high-importance notification channel, plus the runtime lock-screen hold described above. Android grants and can revoke the full-screen intent permission, so the code degrades to a heads-up card with the same actions instead of failing. The [Android notification guidance](https://developer.android.com/develop/ui/views/notifications/time-sensitive) is a good starting point.
