---
layout: post
title: "Folcadi bots: give your chat a small, scriptable assistant"
description: "Create a bot in the Folcadi app, copy its token once, and drive it from any script with a small HTTP API — in private chats and groups."
tags: showdev, api, bots, python
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-21-folcadi-bot-api.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-21-folcadi-bot-api.png
---

A bot is a chat participant with its own identity: a name, a handle, and a
small inbox. It does not need a phone, a session, or a screen. You create it
inside Folcadi, copy a token once, and then any script you can run — on a
laptop, a Raspberry Pi, or a server — can read its messages and answer them.

This is the first release of that feature. It covers private chats and groups,
text messages, and a small HTTP API with long polling. No SDK, no webhook URL,
and no public server required to get started.

## Create a bot in the app

Open **More → Bots** and tap **+**.

- **Username** — lowercase, ends in `bot`, for example `weatherbot`. This is the
  handle people use to find and mention the bot.
- **Name** — the display name shown in chats.
- **Description** — one or two lines shown on the bot's profile.

The next screen shows the **API token**. It is displayed exactly once. Copy it
and store it in your script's environment, not in the source code. The token is
the only credential the bot has; the app keeps a prefix so you can recognise it
later, but it can never show the secret again.

If the token leaks, open the bot and choose **Regenerate token**. The old one
stops working immediately and a new one is shown once. You can also delete the
bot from the same screen; its token is revoked and its group memberships are
cleaned up.

From the bot screen you can also start a private chat. A bot cannot open a chat
with a person first — the person has to open it.

## The API at a glance

```
Base URL:  https://api.folcadi.com/v1/bot
Auth:      Authorization: Bearer <token>
Transport: HTTPS, POST, JSON
```

Every response uses the same envelope:

```json
{ "ok": true, "result": { } }
```

and errors look like this:

```json
{ "ok": false, "error_code": 403, "description": "bot is not a member of this chat" }
```

The first release exposes five methods:

| Method | What it does |
|---|---|
| `getMe` | identity check: id, username, name, description |
| `getChats` | every private chat and group the bot is in |
| `getChat` | one chat, with member count and the bot's role |
| `getUpdates` | long-poll for new messages |
| `sendMessage` | send a text message |

Calls are ordinary POSTs with a JSON body. A minimal check:

```bash
curl -sS https://api.folcadi.com/v1/bot/getMe \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" -d '{}'
```

```json
{
  "ok": true,
  "result": {
    "id": 123,
    "username": "weatherbot",
    "name": "Weather",
    "description": "Daily forecasts",
    "picture_url": null
  }
}
```

## Reading messages without a public server

`getUpdates` is a long poll. You send the cursor you have already processed and
the method waits up to 25 seconds for the next message:

```json
{ "offset": 0, "limit": 100, "timeout": 25 }
```

It returns a page of updates and the cursor for the next call:

```json
{
  "ok": true,
  "result": {
    "updates": [
      {
        "update_id": 31804,
        "type": "message",
        "chat_id": 77,
        "chat_type": "private",
        "chat_title": "Alice",
        "peer_user_id": 41,
        "from_user_id": 41,
        "from_name": "Alice",
        "message_id": 9142,
        "text": "hello bot",
        "is_command": false,
        "command": null,
        "mentions_bot": false,
        "reply_to_message_id": null,
        "reply_to_bot": false,
        "created_at": "2026-11-21T10:12:33+00:00"
      }
    ],
    "next_offset": 31805
  }
}
```

Two rules keep the cursor correct:

1. **Use `next_offset` from this response.** It advances past the updates you
   just received.
2. **Never use a message id from `sendMessage` as an offset.** Update ids and
   message ids are different things. A bot that polls, sends a reply, and then
   adopts the reply's message id as its cursor can silently skip a user message
   that arrived in between.

If there is nothing new, the call waits and then returns an empty list with the
same offset. That is normal. Poll again.

## Sending a message

```json
{
  "chat_id": 77,
  "text": "Sunny tomorrow, light wind from the west.",
  "reply_to_message_id": 9142
}
```

`reply_to_message_id` is optional. `chat_id` is the conversation id from
`getChats` or from an update — for a private chat it is not the same as the
person's user id, so use the ids the API gives you.

Text is limited to 4096 characters. Messages go through the same delivery path
as a human message, so the recipient gets the normal chat preview, unread count
and notification.

## A tiny echo bot

This is enough to see the whole loop: read, decide, reply.

```python
import os
import requests

BASE = "https://api.folcadi.com/v1/bot"
HEADERS = {"Authorization": f"Bearer {os.environ['FOLCADI_BOT_TOKEN']}"}


def call(method, **body):
    response = requests.post(
        f"{BASE}/{method}", headers=HEADERS, json=body, timeout=40
    )
    payload = response.json()
    if not payload["ok"]:
        raise RuntimeError(payload["description"])
    return payload["result"]


offset = 0
while True:
    page = call("getUpdates", offset=offset, timeout=25)
    for update in page["updates"]:
        if update["type"] != "message" or not update.get("text"):
            continue
        call(
            "sendMessage",
            chat_id=update["chat_id"],
            text=f"echo: {update['text']}",
        )
    offset = page["next_offset"]
```

Run it with the token in the environment, open a chat with the bot in the app,
and type. The bot answers. Nothing else is required: no port forwarding, no
public URL, no webhook registration.

## Groups

A bot can be a member of a group. Open the group's info screen and use the
**robot button** next to *Add members*. Pick one of your bots and choose what it
may read:

- **Only mentions and commands** — the bot receives `@username` mentions,
  `/commands`, and replies to its own messages. This is the default.
- **All messages** — the bot receives every human message in the group.

The choice is deliberate, and it is visible in the group. A bot can never read
messages that were sent before it joined, and it never receives another bot's
messages.

If the group is announce-only, a normal member cannot post — including a bot.
Promote the bot to group admin in the member list, and it can reply. Promotion
also gives the bot full visibility, which is why it is an explicit action.

## What the first release does not do

It is better to be precise than to promise everything at once:

- **Text only.** Images, files and voice messages are not part of this release.
- **No bot-to-bot messages.** A bot never receives another bot's output, which
  removes the easiest way to build an accidental infinite loop.
- **No encrypted chats.** Bots work in ordinary private chats and groups.
- **No webhooks yet.** Long polling is the supported delivery mode; it needs no
  public endpoint and works from a home network.
- **A small bot quota per account.** It is a starter allowance, not a limit on
  what one bot can do.

## Where to find everything

- **In the app:** **More → Bots** is the source of truth. It shows your bots,
  the token prefix, when the token was last used, and the buttons to regenerate,
  delete, or start a chat.
- **API base:** `https://api.folcadi.com/v1/bot`.
- **This article:** the quick reference for the five methods and the update
  shape.

The app is available on Android, Windows and Linux. Create a bot, copy the token
once, and run the echo script. The first message is the hard part; after that it
is just a loop.

## FAQ

**Do I need a server?**
No. Long polling works from any machine that can make outbound HTTPS requests.
A laptop that sleeps will miss polls until it wakes, but nothing is lost — the
next call returns the messages after your cursor.

**Can the bot message me first?**
No. A person opens the private chat first. This is the main anti-spam rule.

**Can I use the same bot in several groups?**
Yes, within a small per-bot group allowance. Each group has its own privacy
mode.

**What happens if I lose the token?**
Open the bot and regenerate it. The old token dies immediately and the new one
is shown once.

**Can the bot read messages from before it joined?**
No. The history cutoff is set when the bot is added.

**Can a bot own a group?**
No. A bot can be a member or an admin, never the owner.

**Can I run two bots in one group?**
Yes. In the default privacy mode each bot only reacts to its own mentions,
commands and replies, so they do not step on each other.

---

**Get Folcadi** — a language-exchange messenger with live translation.

- **Official website:** [folcadi.com](https://folcadi.com)
- **Android:** [Google Play](https://play.google.com/store/apps/details?id=com.folcadi.app)
- **Windows and Linux:** [desktop downloads](https://github.com/FrankMilvus/folcadi-releases/releases/latest)
- **Articles:** [write-ups and covers](https://frankmilvus.github.io/folcadi-articles/)
