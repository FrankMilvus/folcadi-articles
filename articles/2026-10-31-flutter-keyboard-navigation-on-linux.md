---
layout: post
title: "Keyboard navigation in Flutter on Linux: one modifier, cursors"
description: "Keyboard navigation in a Flutter app on Linux, tested on Arch + niri: an Alt or Ctrl modifier, cursors everywhere, and the gaps still left."
tags: flutter, linux, accessibility, folcadi
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-31-flutter-keyboard-navigation-on-linux.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-31-flutter-keyboard-navigation-on-linux.png
---

Keyboard navigation in Flutter sounds like a solved problem until you try to add it to a real app. The framework gives you `Shortcuts`, `Actions`, and a focus tree, and on a form with a handful of fields that is genuinely enough. Then you open a chat, and the thing you want to activate is a message bubble that is not a focusable control, the list under it is reversed, an overlay is anchored to a row that may scroll away, and the user may be typing at the same time.

That is the situation in Folcadi, a language-exchange messenger with a Flutter client. This post is not a celebration of complete keyboard parity. It is a description of the model we settled on, the two rules that keep it honest, and the parts that are still mouse-only.

## The first attempt was a pile of letters

The earliest version was the obvious one. Each screen got letter chords: `Alt+S` for Settings, `Alt+M` for Saved Messages, `Alt+F` for Feedback, and so on. It worked, and it was discoverable, because the tiles could show their own hint.

It also did not scale. Every new row wanted a letter, the letters started to collide across screens, and the help text grew into a list nobody could remember. Worse, the model implied that the keyboard was a second set of buttons rather than a way to move through the interface.

The version that survived is smaller:

- one modifier, `Alt` by default, with `Ctrl` as an option for keyboards or browsers that claim `Alt`;
- plain keys stay plain, so `j`, `k`, `r`, and `d` keep typing normally;
- navigation is a cursor that moves and activates, not a shortcut per item.

That last point is the whole design. Lists, menus, confirmation dialogs, and the chat message list all use the same interaction: move the highlight, press Enter, press Escape to go back. New rows do not need a new letter; they become reachable the moment they are added to a list.

## Resolve the modifier at match time

The modifier is a user setting, not a compile-time constant. Users pick `Alt` or `Ctrl` in Settings → Shortcuts, and the choice is stored locally and synced with the rest of their settings. Handlers read it when a key arrives, so changing it takes effect immediately, without rebuilding every screen.

The helper is deliberately small:

```dart
bool get _useCtrl => hotkeyModifierNotifier.value == HotkeyModifier.ctrl;

KeyboardShortcut mainHotkey(LogicalKeyboardKey key) =>
    KeyboardShortcut(key: key, alt: !_useCtrl, control: _useCtrl);

bool matchesMainHotkey(KeyEvent event, LogicalKeyboardKey key) =>
    mainHotkey(key).matches(event);
```

Everything else — handlers, help labels, the in-app cheat sheet — goes through helpers like these. When a chord is displayed, it is formatted from the same source of truth, so switching to `Ctrl` changes the labels too.

## Cursors are the unit of navigation

`KeyboardListCursor` is the reusable piece. A page supplies the number of rows, an activation callback, and a builder that highlights the row at the current index. The widget owns the index, the raw key handler, and scrolling the highlighted row into view.

The important part is not the widget. It is that the row is built as a pair — the widget and the action — and the tap handler and the Enter handler call the same function. The keyboard path cannot drift from the touch path, because there is only one path.

The same cursor covers the More tab, Settings, contacts, notifications, pinned messages, blocked users, the account switcher, and the recipient pickers for forwarding, sharing, and adding members. Confirmation dialogs follow the same rule: `H`/`K` and `L`/`J` move between the buttons, Enter runs the focused one, Escape cancels. There are no `Y`/`N` letter chords, because a dialog is just another two-row list.

Pages that have nothing to activate — Help, profiles, device settings, a long settings sub-page — get a scroll scope instead: `J`/`K` for a small step, the arrow keys for half a page. A page gets a cursor or a scroll scope, never both. Menus need a highlight; switches and sliders need movement.

## The chat is where the model gets tested

The chat was the hardest surface. It has bubbles rather than rows, a reversed scroll view, media that can be tall, and a composer that wants every plain letter.

The message cursor moves with `J`/`K`. The first press lands on the newest message, movement clamps at both ends, and date separators and system chips are skipped. `Space` opens the actions menu for the highlighted message, and that menu is a two-dimensional grid: `H`/`L` move across, `J`/`K` move down, Enter activates. The quick-emoji row is row zero, so reactions are reachable without a detour through the action list.

The most recent change removed a round trip. `R`, `E`, and `D` now act directly on the highlighted message: reply, edit, delete. The menu still offers the same actions, and the eligibility rules are shared with the menu rather than duplicated, so a message that cannot be edited cannot be edited by either path. When the highlighted message is not yours, the keys do nothing.

`R` was the interesting case, because it already meant "record a voice message" in the composer. Two meanings for one key is how a keyboard model rots. The fix was to make the two modes mutually exclusive: moving the cursor blurs the composer, and focusing the composer — by tap, by `I`, or by choosing Reply or Edit — drops the cursor. With a highlight, `R` replies; with no highlight, `R` records. The state of the screen tells you which one you will get.

Half-page scrolling moved to the arrows at the same time. `D` and `U` had been the page-scroll chords, which was fine until `D` became Delete on the highlighted message. The arrows were unclaimed and are easier to explain. The chat's list is reversed, so the scroll scope flips the sign by axis direction: "down" always means visually down.

The chat also binds `F` for in-chat search, `C` for the call chooser, and `B` for the translation mode. Each calls the exact callback the app-bar button calls.

## Escape should cancel before it navigates

The back button is destructive if it leaves the screen while something is half-finished. If you have started a reply, staged a photo, or begun a recording, the first Escape should undo that, not walk you out of the conversation.

The chat's back handler now vetoes navigation while any transient state is active, and unwinds it in a fixed order: message actions overlay, multi-select, search panel, app-bar menu, attach bar, voice recording, edit mode, reply bar, staged image or album, expanded editor. Only when none of those is active does Escape leave the chat.

That one change made the rest of the hotkeys feel safe. Escape is "cancel" for a reply, an edit, or a recording, and Escape is still "back" when there is nothing to cancel. The rule is easy to state and, once implemented, easy to keep — the new `E` and `D` actions inherited it for free.

## The focus problem nobody warns you about

Flutter's `HardwareKeyboard` calls every registered raw handler before the focus system gets the event. Returning `true` from a handler does not stop the focus tree from dispatching it afterwards.

This is not a theory. In a review pass over an earlier keyboard sweep, we found that pressing Enter with a cursor set could run the highlighted row *and* a button that happened to own focus — for example, opening two stacked confirmations in the account switcher. The fix is a rule, not a workaround: if a control inside the same route already owns focus, the cursor yields and Enter belongs to that control. When nothing in the route owns focus, the cursor takes it.

The same property — every raw handler runs — creates the second rule: one chord, one owner. Two features must never bind the same combination while both are live. A sweep caught exactly that when `Alt+X` was bound both to "remove this account" in the switcher and to "ignore this call" on the incoming-call ring. Ignore moved to `Alt+N`. The ring's keys also avoid the letters used by the cursors underneath, because a ring is mounted above every route and shares the focus scope of whatever page is visible.

These are the kinds of bugs that pass a clean analyzer and a green test suite. The behavioural review found them; the tests were written afterwards.

## What this was tested on

Everything described here has been developed and tested on **Arch Linux with the niri compositor**. The key handling itself is ordinary Flutter — `HardwareKeyboard`, `ShortcutScope`, and raw handlers, with no compositor-specific code — so there is no reason it should be Arch- or niri-specific. But that is the only combination that has been exercised end to end so far.

Other Linux desktops, X11 sessions, Windows, macOS, and Android with a physical keyboard are expected to behave the same, and the widget tests run on Flutter's standard test platform. They are not verified on real hardware yet. If you try the app on another setup and a chord is inert, fires twice, or fights a desktop shortcut, that report is genuinely useful — it is the kind of gap this post is trying to be honest about.

## What is not done yet

It would be easy to write this post and imply that Folcadi now works fully without a pointer. It does not, and I would rather say so plainly.

- **Some surfaces are still pointer-first.** Gestures that have no obvious key equivalent — several drag and swipe interactions, for example — are reachable only by touch or mouse today.
- **Not every overlay is fully navigable.** The main ones are; a few secondary surfaces still need a pointer or Tab.
- **There is no complete focus traversal.** The app has a cursor model, not a full tab-order contract. A screen reader user and a keyboard-only user need different things, and the accessibility work is not the same project as the hotkey work.
- **Web is the weak platform.** Browsers claim several modifier combinations before the app sees them. The hotkeys are designed for desktop and for Android with a physical keyboard, including environments that map a PC keyboard into Android; on the web they are best-effort.
- **Mobile is for external keyboards.** The chords assume a hardware keyboard. On a phone with only a soft keyboard, the touch UI remains the primary interface.
- **Only one desktop setup is verified.** Development and testing happen on Arch Linux with niri; other desktops and operating systems should work by construction but have not been exercised on real hardware.
- **Documentation drifts.** Every new surface has to register its groups with the cheat sheet and the in-app help. If that step is skipped, the feature exists but is undiscoverable.

The direction is to keep closing gaps in that order: make the common flows complete, make the state cancellation predictable, and only then chase breadth.

## How to discover what works

Press `F1` anywhere. The cheat sheet is assembled from the scopes that are actually active on the current screen, so a plain page shows a few rows and a chat adds its own. The same chords are documented under More → Help, and the labels follow whichever modifier you chose.

If you use a keyboard with the app and find a place where you reach for the mouse, that is useful feedback. The gaps above are known; the one you hit is the one worth fixing next.

## FAQ

### Which platforms has this been tested on?

Arch Linux with niri. The handlers are plain Flutter and the widget tests run on the standard test platform, so other Linux desktops and Windows or macOS should work, but they have not been verified on real hardware yet. Web is limited by browser shortcuts, and mobile without an external keyboard still relies on touch.

### Can I use Ctrl instead of Alt?

Yes. Settings → Shortcuts offers both. The choice is stored locally and synced like other settings, and every hint and help label updates to match.

### Why not use plain letters, like a terminal?

Because the app has text fields everywhere. A chat composer, a search box, and a filter field all need ordinary letters. Requiring a modifier keeps typing and navigation from competing for the same keys.

### How do I cancel something I started with the keyboard?

Press Escape. It unwinds the current transient state first — a reply, an edit, a recording, a selection — and only leaves the screen when there is nothing left to cancel.
