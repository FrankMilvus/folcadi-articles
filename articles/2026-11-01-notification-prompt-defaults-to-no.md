---
layout: post
title: "Pre-ticked boxes are not consent: notification prompts default to off"
description: "A one-time notification prompt where every channel starts off: separate purposes, tri-state consent, a fail-open call gate, and one-click unsubscribe."
tags: flutter, ux, privacy, mobile
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-11-01-notification-prompt-defaults-to-no.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-11-01-notification-prompt-defaults-to-no.png
---

Notification permissions are usually asked for badly. The app appears for the first time, and before the user has read a single message it shows the operating system's permission dialog. If they say no, the prompt is gone for good and the app quietly decides that notifications are not for this person. If they say yes, the app now has a blank cheque for message alerts, calls, reminder emails, and whatever marketing is invented next.

We shipped the opposite last week in Folcadi, a language-exchange messenger with a Flutter client. After login, once, the app shows its own preference sheet with four channels. Every switch starts off. Nothing is pre-ticked, nothing is bundled, and "Save" with everything off is a complete, accepted answer.

This post is about why the defaults are the whole product decision, and about the code shapes that follow from taking consent seriously: tri-state columns, a fail-open call gate, and an unsubscribe link that works without logging in.

## The prompt we replaced

The old app had three problems, and they are the usual three.

The first was **timing**. The startup path requested the OS notification permission for every user, whether or not they had ever wanted a notification. On Android 13 and later that dialog is a one-shot resource: a reflexive "don't allow" during launch is hard to undo, and a lot of users never find the settings page to reverse it.

The second was **bundling**. One "notifications" concept covered message alerts, incoming-call ringing, a reminder email for unread chats, and product news. Those are four different offers with four different costs to the user. A person who wants to know about a missed call may not want a marketing email, and an app that conflates them has decided that for them.

The third was **the data model**. There was no record of anybody's preference, because there was no preference to record. The server mailed and pushed until something errored, and the only way to stop it was to silence the app at the OS level.

## Four purposes, four switches

The new sheet is one themed dialog with four rows and a matching set of toggles in Settings → Notifications:

| Choice | Starts | What it gates |
|---|---|---|
| Message notifications | off | Push alerts for new messages; turning it on asks the OS permission and registers this device |
| Incoming calls | off | Ringing for background calls; account-level, not per device |
| Email reminders | off | One reminder email if messages stay unread for days |
| Product news and offers | off | A separate marketing consent; nothing is sent under it yet |

The copy is plain, and each row says what it does rather than what it is called: "Ring for incoming calls on your devices", "Email me if messages stay unread for a few days". The footer states the deal in one line: we only send what you select, and every reminder email carries a one-click unsubscribe link.

Separating the channels is not just politeness. It is the difference between consent that is **specific** and consent that is a blanket. A single switch forces a user who wants call alerts to accept marketing to get them, or to refuse everything to avoid marketing. Both outcomes are worse than four honest questions.

## Why pre-ticked is not consent

In the EU, consent under the GDPR and the ePrivacy rules has to be freely given, specific, informed and unambiguous. Silence, inactivity, and pre-ticked boxes do not count. That is not an interpretation; the Court of Justice said it directly in *Planet49* (C‑673/17, 2019), a case about pre-ticked cookie checkboxes. Ticking a box for someone, or treating "they did not untick it" as agreement, fails every part of that test.

I am not a lawyer and this is not legal advice. But the engineering consequence is simple, and it holds even in places where marketing consent is not a legal question at all: **a default is a statement about who is making the decision.** If the switch is on before the user arrives, the app made the decision and the user is only correcting it. If the switch is off, the user decides. That is also the only version that survives contact with an audit, an app-store review, or a reader asking "how do I stop this".

Marketing email is the strictest case. It gets its own switch, its own consent stamp, and its own unsubscribe path, even though we are not sending any yet. Asking for consent before you need it is fine. Assuming it is not.

## Consent is a tri-state, not a boolean

Here is the part that sounds like pedantry and is not. A boolean column cannot tell the difference between "never asked" and "said no". If you treat both as false, then adding the prompt silently turns off call ringing for every existing user who has not opened the app since the release — a change nobody agreed to, discovered when somebody misses a call.

So the columns that represent a **choice** are nullable, and the third state is meaningful:

```python
#   None  -> no choice recorded yet: keep the previous behaviour
#   True  -> opted in
#   False -> explicitly opted out
class NotificationConsent(Base):
    push_alerts: Mapped[bool | None] = mapped_column(nullable=True)
    call_alerts: Mapped[bool | None] = mapped_column(nullable=True)
    email_reminders: Mapped[bool] = mapped_column(default=False)
    product_news: Mapped[bool] = mapped_column(default=False)
```

Email reminders and product news default to `false` because there is no older behaviour to preserve: they are new channels, and new channels start off. Push and call alerts default to `NULL`, because they already worked, and a user who has not been asked must keep what they had. Only an explicit `false` suppresses anything:

```python
async def opted_out_of_calls(db, recipients: list[int]) -> set[int]:
    rows = await db.execute(
        select(NotificationConsent.user_id).where(
            NotificationConsent.user_id.in_(recipients),
            NotificationConsent.call_alerts.is_(False),   # explicit no only
        )
    )
    return set(rows.scalars().all())
```

That single `.is_(False)` is the difference between honouring a "no" and inventing one.

## One OS permission, hidden behind a product choice

There is only one notification permission at the operating-system level, and both message alerts and call ringing need it. Asking for it before the user has expressed an interest wastes the one shot you get.

The sequence we use is: the user picks channels in our sheet, and only if message alerts or calls are switched on do we ask the OS. If the OS says no, the switches go back to off and the app explains it instead of pretending to save:

```dart
var pushChoice = supportsPush ? _push : null;
var callsChoice = _calls;
var denied = false;

if (supportsPush && (pushChoice == true || callsChoice)) {
  final granted = await notificationService.requestOsPermission();
  if (!granted) {
    pushChoice = false;
    callsChoice = false;
    denied = true;
  }
}
```

The message after a denial is deliberately undramatic — "Notifications are blocked in system settings. You can enable them there later." — because a person who declined twice should not be scolded by an app they just installed.

The other half of this is unregistering. When a device ends up with no push channel on, its push token is deleted for that account, so "off" means off rather than "off in the UI, still buzzing".

## Fail open where silence is dangerous

Preference lookups can fail. Databases time out, replicas lag, a migration lands mid-request. The question is what the failure means for the user.

For a marketing email, failing closed is right: if we cannot prove consent, we do not send. For an incoming call, the same logic is catastrophic. If the preference query throws and the code treats that as "not allowed", a phone stops ringing because of an unrelated hiccup. That is a missed call caused by a product decision about consent.

So the call gate fails open, explicitly:

```python
try:
    rows = await db.execute(select(...).where(...))
    return set(rows.scalars().all())
except Exception:
    # A preference lookup must never be the reason a call stays silent.
    logger.warning("call-alert preference lookup failed", exc_info=True)
    return set()
```

The asymmetry is the point, and it is worth writing down next to the code: **consent gates should fail closed for growth and open for safety.** Marketing, newsletters, and re-engagement features can safely assume the answer is no. Calls, security alerts, and anything a user would call urgent cannot.

One more detail from that gate: cancel pushes are still sent to a user who turned call alerts off. Otherwise a device that was already ringing when the preference landed would keep ringing with no way to dismiss it.

## "Not now" must not count as an answer

The sheet is shown once, and the server records both that it was completed and which version of it was shown. That gives us a cheap way to ask again if the choices materially change: bump the version, and the prompt returns once for everyone.

Dismissing is different from deciding. "Not now" closes the sheet without completing it, and the prompt can come back on a later launch — at most once per 24 hours, tracked locally per account. The cap matters: a preference that is genuinely useful deserves to be asked again later; it does not deserve to be asked on every cold start, which is how prompts become wallpaper.

The sheet also refuses to cover the wrong moment. If the app opened a chat or a call from a notification or a deep link, the prompt waits for a route where nothing is in progress, and gets offered on the next launch instead. Interrupting the one thing the user opened the app for is how you teach them to dismiss you.

There is a small implementation detail in the versioned-prompt model that is worth copying: the server is the source of truth for "this person has answered", and a local timestamp is only there to stop the sheet reappearing before the next launch. Clearing app data loses the local timestamp, not the decision.

## Push is per device, calls are per account

The same sheet runs on a phone, a desktop client, and anything else signed into the account, and the four rows do not all mean the same thing on each.

Message notification is per device: a phone has a token, a desktop may have none, and a preference set on the phone should not be silently rewritten by a laptop that cannot receive push at all. So the desktop hides that row and reports "no opinion", which is the `NULL` state from earlier. The Calls choice is account-level, because a call should ring wherever the person is.

Getting this wrong is easy and invisible in testing. A desktop client that sends `false` for a channel it cannot even show looks like a clear preference in the database and a bug report in the making: "everything is off on my phone, and I never touched it".

## Unsubscribing without logging in

Every channel has to be as easy to leave as it was to join, and email is the one where that is hardest, because the user may be reading it in a client that has no idea who they are.

Reminder emails therefore carry a signed link that needs no session, a visible unsubscribe link, and the standard one-click unsubscribe headers. Opening the link flips the flag off. It is the same column the in-app toggle writes, so the two cannot disagree, and the same consent stamp gets a new timestamp — a withdrawal is a decision too and deserves a record.

In the app, Settings → Notifications exposes all four channels with the same write path as the sheet. Changing your mind does not require finding the original prompt again.

## What this costs

Honesty section: opt-in rates after this change are lower than they were when the permission was requested automatically. That is the intended result, not a bug.

The trade we made is a smaller list of people who actually want the alerts, in exchange for a list that means something. A push sent to a user who reflexively tapped "allow" during onboarding is worth very little; a push sent to someone who turned the switch on is worth something, and a push sent to someone who turned it off is a reason to uninstall. The same logic applies to the reminder email: it is opt-in, capped to one a week per person, and quiet for anyone who has been active.

We do not have retention numbers to show for this yet — the change is days old, and claiming a measured win would be inventing one. What we can say is that the support-shaped problem is gone: there is now one place that answers "why is this app emailing me", and it is the same place that stops it.

## A checklist you can copy

If you are about to ship a notification prompt, these are the parts that mattered most:

- **Ask after the user has seen something worth notifying them about**, not during the first launch animation.
- **Split the purposes.** Message alerts, calls, reminder email, and marketing are four offers, not one.
- **Start every switch off**, and make "all off" a real, saved answer.
- **Make consent tri-state.** "Never asked" must not behave like "no".
- **Ask the OS last**, once, after a product-level yes. Revert the switch and explain if the OS declines.
- **Write down which gates fail open.** Calls and security alerts keep working when the preference lookup breaks; marketing does not.
- **Version the prompt** so a material change can be asked about once, without becoming a nag.
- **Make leaving as easy as joining**, including a link that works without a session.

None of this is exotic. It is mostly the decision to let the user answer the question instead of answering it for them, and then to build the data model that can remember the difference between "not yet" and "no".

## FAQ

### Is a notification prompt legally required?

Not in general, and this post is not legal advice. What is required in the EU is that consent — especially for marketing email — is freely given, specific, informed and unambiguous, which rules out pre-ticked boxes and silence. Separating the channels and defaulting them off is how you satisfy that while still asking for the alerts that make the app useful.

### Why not just ask for notifications on the first launch?

Because the OS permission is a one-shot resource and the first launch is the worst moment to spend it. A user who has not received a message yet has no reason to say yes, and a reflexive "no" is much harder to undo than a considered "yes" is to earn.

### What happens if someone dismisses the sheet?

Nothing is recorded as a decision. The sheet can be offered again on a later launch, at most once per 24 hours, and Settings → Notifications is always available. Only pressing Save writes the completion stamp.

### Do existing users lose call ringing when they update?

No. The push and call preferences start empty (`NULL`), which keeps the previous behaviour until the person actually answers the prompt. Only an explicit "off" suppresses anything.

### Does turning off message notifications still leave the account reachable?

Yes. In-app calls still arrive while the app is open and connected, and unread counts still work. The switch controls push delivery, not the account.

### What does the reminder email contain?

Only a nudge that messages are waiting, with a link back to the app and a one-click unsubscribe. It never includes message content, and for encrypted chats it does not name the sender or the conversation either.
