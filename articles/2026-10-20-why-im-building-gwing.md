---
layout: post
title: Why I'm building GwinG, a language exchange app for awkward hellos
description: "I started GwinG after trying language-exchange apps and realizing the hard part wasn't finding people. It was keeping the conversation alive."
tags: languageexchange, flutter, showdev, buildinpublic
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-20-why-im-building-gwing.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-20-why-im-building-gwing.png
---

## I'm not a language influencer

I'm FrankMilvus, and I build **GwinG**, a language exchange app. No, I'm not fluent in seven languages, and I don't have a 30-day method to sell you. I'm a developer who wanted to talk to people and got annoyed at how much work the apps made it feel like.

GwinG is the thing I wish I had about five years ago, when I installed a couple of the big language-exchange apps, found people to talk to, and then realised the apps were much better at matching than at the conversation that was supposed to come after.

The idea is roughly five years old. The current codebase is not. I started the version you can install today in July, and the repo is already at almost 900 commits. That number sounds impressive until you notice how many of them say "fix" or "release" and nothing else.

Up front: GwinG is not trying to be the biggest messenger in the world. It's a small product for a specific moment. The moment you want to say something to someone who speaks another language, and you don't want to spend ten minutes translating it in your head first.

## The part most language exchange apps skip

Five years ago I wanted to find someone to speak with. Not a course. Not a flashcard deck. A person.

I tried a couple of the usual apps. They did the matching part fine: here's a profile, here's a language, here's a button. Then came the actual conversation, and that part was on me.

Finding a person is a directory problem. Having a conversation is a product problem. A lot of apps stop after the directory.

That's the gap I kept running into. The first message is easy enough. It's the fifth one that gets awkward. You run out of things you can say with the vocabulary you have, the other person switches to English to be helpful, and suddenly the language exchange is over and you're just two people chatting in the same language again.

## What I wanted instead

I wanted the app to carry some of that weight.

If I have to copy a message, paste it into a translator, copy the result back, and hope the tone survived, I'm not having a conversation anymore. I'm doing data entry with extra steps. So GwinG puts translation inside the chat. You write in your language, the other person reads in theirs, and the original is always one tap away.

Voice messages work the same way. Transcription turns them into text you can read, search and translate. That matters more than it sounds. A voice note in a language you're still learning is basically a locked box. Text makes it something you can actually work with.

The point isn't to hide the other language. It's to give you a ladder. You read the translation, then you look at the original, and slowly you need the translation less. That's the whole trick. It's not magic, and it doesn't replace the awkward part. It just gives you something to hold onto while you get through it.

## What GwinG actually is

The short version: a messenger. Chats and groups, voice and video calls, photos, voice notes, video, files, replies, reactions, pins, search, friends, presence. The usual stuff, plus the translation layer that makes the exchange part possible.

Under the hood it's [Flutter](https://flutter.dev) on the client and [FastAPI](https://fastapi.tiangolo.com) with PostgreSQL on the server. WebSockets for realtime, WebRTC for calls, object storage for media, and small services for speech and translation. Android, iOS, Windows and Linux. The code is mostly mine. I'm the person who gets to fix the weird desktop bug at 1am, which is a very effective way to learn desktop packaging.

Not because I'm some 10x developer. Mostly because the app is small enough to hold in my head, and I'm stubborn.

## The team is basically a group chat

There are a few of us. We're spread out. Spain, South Korea, a few other places. So the app gets built across time zones whether we plan for it or not.

Between us we speak English, Spanish and Russian, and we're currently poking at Korean and German. Which means we're not guessing what it feels like to be the person who understands half the conversation and smiles anyway. We've all been that person.

That's also why a lot of the product decisions are boring on purpose. If a feature only works when everyone is already fluent, it's not a language-exchange feature. It's a feature for people who don't need the app.

## The name is not the product

People ask what GwinG means. Honest answer: it's a project name.

We don't care about it the way you care about a child's name. If the app is good, the name becomes good. If the app is bad, no name saves it. We already changed it once, from tonGues, and the thing that mattered was that nothing underneath changed. Same accounts, same conversations, same keys. The name is a label on the box. The box is the product. If you want the whole rename story, [it's here](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-18-why-we-renamed-tongues-to-gwing.html).

What I like about GwinG is that it doesn't pretend to mean anything in particular. It's five letters with a capital at each end. It's ours. That's enough.

## "Getting better" is the whole roadmap

The line I keep coming back to is this: we're building a platform that lets people get better.

Not just better at grammar. Better at the messy part. Saying something before you're ready. Asking someone to repeat themselves without dying inside. Laughing at the fact that you just used the wrong word for "chicken" in front of a stranger. Coming back the next day anyway.

Fluency is a long game. Confidence is a shorter one. We can help with the shorter one.

So no, we're not promising you'll be fluent in 30 days. Nobody can promise that. We're trying to make the first message less scary and the tenth one easier. That's the whole pitch. It's not a growth hack, and it doesn't fit in a billboard.

## What we're working on now

We're still in the unglamorous phase.

Calls that connect on the first try. Messages that don't get lost when the network blips. Translation that doesn't sound like a robot reading a legal document. Desktop builds that behave like desktop apps instead of a phone app in a big window. The list is long, and almost none of it makes a good demo.

That's also why I write everything up. The good changes, the ugly fixes, the ones that looked simple and turned into a three-day detour. You can find the longer version of the project in [the overview post](https://frankmilvus.github.io/folcadi-articles/articles/2026-10-11-what-is-tongues-language-exchange-messenger.html), or just follow the articles as they come out.

## Where to find us

The code is private for now, but the decisions aren't. If you want to follow along, argue with a design choice, or tell me the app is missing the one feature you actually need, here's where:

- **GitHub:** <https://github.com/FrankMilvus>
- **dev.to:** <https://dev.to/frankmilvus>
- **Telegram:** <https://t.me/gwingorg>
- **Articles:** <https://frankmilvus.github.io/folcadi-articles/>

Say hi. I read everything, including the messages that start with "this is probably a stupid question." Those are usually the good ones.

— FrankMilvus
