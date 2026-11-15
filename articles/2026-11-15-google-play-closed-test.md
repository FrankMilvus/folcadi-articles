---
layout: post
title: "Google Play's 14-day closed test: the gate belongs to the app"
description: "Twelve testers for fourteen continuous days before production access — why the clock is a roster problem, and how to plan a window that does not break."
tags: android, buildinpublic, testing, googleplay
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-15-google-play-closed-test.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-15-google-play-closed-test.png
---

Our closed test for Folcadi finished this week. The testing was the easy part. The scheduling was not.

Google Play asked for a closed test with at least twelve testers held for fourteen continuous days before it would consider the app for production access. We had assumed our existing developer account would count for something here. It did not: the requirement is attached to the app being uploaded, so a listing that has never shipped starts its fourteen days from zero no matter how old the account behind it is.

That single misunderstanding cost us planning time. The rest of this post is about the part that actually decides whether the window completes.

## Twelve testers, fourteen continuous days

The number is not the hard constraint. Twelve people who want to try an app is a weekend of asking. The hard constraint is the word *continuous*.

The window is a calendar measurement, not a volume measurement. It does not care how many hours of testing happened, how many crashes were filed, or how thorough the QA pass was. It asks whether the roster stayed at or above the minimum, day after day, from the start date to the end date.

Which means the thing you are really choosing when you begin is not a testing plan. It is a fourteen-day period you are confident you can keep staffed.

## The roster is the deliverable

Once we started treating the tester list as the artifact instead of the test plan, the shape of the job changed.

We built the roster in two layers. The core was our own team and the developers building the app — people who already open it daily, on real phones, with real accounts. They were the most useful testers by a wide margin, and just as importantly they were still there on day twelve when the novelty had worn off.

Around that core we added testers from matching pools to reach the required count. That layer behaved less predictably: one source delivered nobody at all over three days and we dropped it. The operational lesson is not "find a better source" but "assume a source will fail, and start the second one before the first one fails."

Two habits made the difference:

- **Keep a buffer above the minimum.** Aim for fourteen or fifteen opted-in testers so that a single drop-out is a smaller number, not a broken window.
- **Stagger the start dates.** Invite the first group, then add a second group a couple of days in. By the time the original roster reaches the second week there are extra opted-in testers behind it, and a person switching phones or going quiet stops being an emergency.

Staggering also gave the matching pools time to fill: some only start their own counting once a pool is complete, so a late addition costs nothing while an early one can stall.

## Write the dates down

The failure mode nobody warns you about is memory. Halfway through, someone asks whether you still have twelve, and the honest answer is that you are counting from a chat thread and two store listings.

Keep one table — invited, joined, left — and update it the day it changes. It is the only way to answer "are we still inside the window?" without guessing, and it is what you want in front of you if the count ever dips.

## What fourteen days actually finds

Given how administrative the gate feels, the window turned out to be more useful than we expected, precisely because two weeks is long enough for boring problems to surface.

Bugs that a single afternoon of testing never catches showed up in the second week: notification behaviour after a device restart, a background queue that only stalls after a day of real chatting, layout problems that need enough messages to scroll before they appear. None of these are hard to reproduce once you know they exist. All of them are invisible in a scripted pass.

We shipped fixes during the window rather than after it. Each build also had to reach testers through the store instead of a downloaded file, so the update path got exercised repeatedly before a single outside user touched the app — a rehearsal that would otherwise cost a production incident.

## A checklist for the next one

- Confirm early whether the requirement follows the app or the account, and plan for the stricter answer.
- Recruit a core of daily users first; they are the testers who stay.
- Recruit second and third sources in parallel, and start them before you need them.
- Set the roster above the minimum and stagger the invitations.
- Track invited/joined/left dates in one place you will actually open.
- Spend the fortnight in daily use, not in one scripted pass.

## The takeaway

The closed test is a formality until about day nine, when it becomes calendar arithmetic. The teams that get through it are not the ones with the best test plan — they are the ones whose roster never dipped.

If you have run one of these recently, I would like to hear what nearly broke your window: a tester disappearing, a pool that never filled, or the wait itself.
