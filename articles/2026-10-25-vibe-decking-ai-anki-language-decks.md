---
layout: post
title: "Vibe-decking: how I use AI to build Anki decks for languages"
description: "Building Anki language decks with an AI chat: a no-code workflow for grammar, vocabulary and reading cards, plus honest time savings."
tags: anki, languagelearning, ai, productivity
published: true
canonical_url: https://frankmilvus.github.io/folcadi-articles/articles/2026-10-25-vibe-decking-ai-anki-language-decks.html
image: https://raw.githubusercontent.com/FrankMilvus/folcadi-articles/main/images/2026-10-25-vibe-decking-ai-anki-language-decks.png
---

I used AI to add 327 Korean grammar and reading cards to my Anki collection. I also used it to update my vocabulary decks without creating a pile of duplicates.

You do not need to know how to code to do this. You do not need a special app, an add-on, or a subscription to a flashcard service. You need three things:

- the AI chat you already use (ChatGPT, Claude, Gemini — whichever you like),
- [Anki](https://apps.ankiweb.net/), the free flashcard app,
- something to learn from: a textbook, a course handout, a grammar reference, or your own existing deck.

The rest is a repeatable loop: **describe the deck, let the AI draft it, review it like an editor, import it.** I call it vibe-decking, because it is the same idea as vibe coding, applied to language learning.

## What vibe-decking has in common with vibe coding

Vibe coding is when you describe an app to an AI, it writes the code, you run it, and you keep fixing what breaks. You can get surprisingly far without being a programmer.

Vibe-decking is the same loop:

| Vibe coding | Vibe-decking |
|---|---|
| You describe the app you want. | You describe the deck you want. |
| The AI writes the code. | The AI writes the cards. |
| You run the app and fix errors. | You study the cards and fix mistakes. |
| Some mistakes show up immediately. | Nothing catches problems for you. |

That last row is the important one. Code usually fails loudly: the app crashes, the button does nothing, the page is blank. A bad flashcard fails silently. It looks perfect, you trust it, and three weeks later you discover you have been practising a wrong translation or a grammar pattern that nobody actually uses.

So the rule is simple: **the AI writes the first draft, you are the editor.** The time saving is real, but it comes from skipping the blank page — not from skipping the review.

## Step 1: Start from something real

Start with one source. It can be:

- a textbook chapter,
- a grammar reference,
- a course handout,
- a list of words you want to learn,
- or your existing Anki deck, if the goal is to improve it.

For the Korean grammar deck, I worked from standard beginner and intermediate grammar. For the reading deck, I worked from the standard pronunciation rules for Hangul, batchim, and sound changes. There was no single book behind those decks, which is exactly why every card needed review afterwards.

Two practical notes:

- **One source at a time.** Mixing a textbook, a website, and a friend's notes makes it harder to tell where a card came from.
- **If there is no source, say so.** The AI can still draft the cards, but you are now relying on its general knowledge. Mark that batch for review before you trust it.

## Step 2: Ask for a sample, not the whole deck

This is the step most people skip. Do not ask for 500 cards on the first try. Ask for 20 or 30, look at them, and fix the style while it is cheap.

A prompt you can copy:

> I am learning Korean. Make 25 beginner grammar cards.
>
> Use five columns: Pattern, Meaning, How it is formed, Two examples, Russian translation.
>
> One grammar pattern per card. Keep the explanation short. Use natural Korean examples with a romanization and an English translation. Do not mix two patterns into one card.

Read the sample and ask yourself:

- Is each card about one thing, or is it a mini lesson?
- Do the example sentences sound like something a person would say?
- Is the translation accurate?
- Is the romanization consistent?
- Are two cards testing the same pattern?

Then tell the AI what to change. "Shorter explanations, one example instead of three, and put the Russian at the bottom" is enough. A good sample saves you from redoing 300 cards.

## Step 3: Generate in batches, by topic

Once the sample looks right, ask for one topic at a time. For Korean grammar, that meant separate batches for:

1. beginner patterns: particles, numbers, negation, basic tenses,
2. intermediate patterns: connectors, cause and contrast, honorifics,
3. reading: Hangul letters, batchim, and sound changes.

Ask for the same columns every time. A spreadsheet is the easiest middle step: paste the table into Google Sheets, Excel, or LibreOffice, check it, and save it as a CSV file. CSV is just a plain spreadsheet file that Anki can read.

Why batches?

- If one batch is bad, you redo one topic, not the whole deck.
- You can review 100 cards in one sitting.
- You notice gaps while the topic is still fresh in your mind.
- You can study beginner cards while the intermediate batch is still being checked.

## Step 4: Import the cards into Anki

In Anki:

1. `File` → `Import`.
2. Choose the CSV file.
3. Pick the deck (or create a new one).
4. Match the columns to the card fields: Pattern, Meaning, Formation, Examples.
5. Import.

The first import takes a couple of minutes. After that it is routine. If you want separate decks per level, import each batch into its own deck — for example, `Korean Grammar::Beginner` and `Korean Grammar::Intermediate`.

You can also make the cards look better later: bold the pattern, add color, or make the examples stand out. That is cosmetic. Get the content right first.

## Step 5: Updating a deck you already have

This is where AI saves the most time, and where most people accidentally create duplicates.

The trick is to show the AI your current deck before asking for more. Export your deck from Anki as a CSV or a plain text file, upload it, and say:

> Here is my current vocabulary deck. I want to add the top 50 Korean verbs. Find the ones I already have and give me only the missing cards, in the same columns. Do not repeat anything that is already in the deck.

I did exactly this with Korean vocabulary. Of the top 50 verbs, 46 were already in my deck. Only four were missing: 걷다, 뛰다, 잃다, and 헤어지다. With the top 50 nouns, 42 already existed and only 8 were new.

Without checking first, I would have added 88 duplicates. A deck full of duplicates is not just annoying — it wastes review time every single day.

"Look at what I have, then give me only what is missing" is the single most useful sentence in this whole workflow.

You can use the same trick to improve existing cards:

> These are my current cards. Find the ones with unnatural example sentences, inconsistent romanization, or two ideas on one card. Give me a corrected version and explain what you changed.

## Step 6: Review like an editor, not like a customer

The AI will produce cards that sound right. Some of them will be wrong. That is normal, and it is your job to catch it.

What I check in a language deck:

- **Meaning and translation.** Does the English or Russian actually match the Korean? A wrong translation is worse than no card.
- **Naturalness.** Is this how people speak, or is it textbook-only Korean that nobody uses?
- **Romanization.** Is it consistent? In my Korean deck, a check found four romanization mistakes, including `오자마자` written as *ojamaso* instead of *ojamaja*, and `닦아요` written as *dakayo* instead of *dagayo*.
- **Pronunciation rules.** For reading cards, does the pronounced form match the standard rules, or did the AI simplify it?
- **One idea per card.** If a card asks for a pattern, its meaning, three examples, and an exception, it is not a flashcard anymore.
- **Your own errors.** If you are not sure about a card, ask a native speaker, check a dictionary, or remove the card until you are sure.

A useful extra prompt:

> Check these cards against the source. List any card that is not supported, any card that repeats another card, and any translation or romanization you are not confident about. Do not rewrite them yet — just flag them.

Then fix the flagged ones yourself. An AI reviewing its own work is not perfect, but it catches more than no review at all.

## What the finished Anki language decks looked like

| Deck | Notes | Cards |
|---|---|---|
| Grammar::01 Beginner | 82 | 134 |
| Grammar::02 Intermediate | 83 | 134 |
| Reading Rules | 59 | 59 |
| **Total** | **224** | **327** |

On top of that, the vocabulary decks got two focused sets: 50 top verbs and 50 top nouns. Most of those were already in the collection, so the AI's job was to find the gaps, not to rewrite what existed.

The grammar cards came in two forms: a meaning card (see the pattern, explain what it means) and a drill card (see an English task, produce the Korean pattern). The reading cards asked "how do you read this?" and showed the pronunciation and the rule on the back.

That variety came from a decision made at the start, not from asking the AI to "make it good".

## How much time does it actually save?

These are rough numbers from my own runs, not a scientific study. Card writing speed varies a lot.

| Job | By hand | With AI |
|---|---|---|
| Korean grammar, 268 cards | 6–9 hours | 2–3 hours |
| Korean reading, 59 cards | 1.5–2 hours | 30–60 minutes |
| Adding 4 missing verbs to a 50-word set | 20–30 minutes | 5–10 minutes |
| Adding 8 missing nouns to a 50-word set | 30–45 minutes | 10 minutes |

The pattern is consistent: **the AI saves the first-draft time, and you still pay the review time.** A card by hand takes one to two minutes. Drafting it with AI takes seconds. Checking it takes maybe ten to twenty seconds. That is where the roughly 3–4× improvement comes from.

It is not 100×, and anyone who tells you it is has not checked their cards. The first draft is no longer the bottleneck. Accuracy is.

## Copy-paste prompts

**Build a grammar deck from a textbook**

> I am learning Korean. I have uploaded [chapter or reference]. Build a study deck one topic at a time. Use the columns Pattern, Meaning, Formation, Two examples, Russian translation. One pattern per card. Use natural examples. Do not invent grammar that is not in the source.

**Build a reading deck**

> Make cards for reading Hangul. Use the columns Prompt, Answer, Examples, Russian. The front shows a letter, syllable, or word. The back shows the pronunciation and the rule that explains it. Include batchim and sound changes.

**Update an existing deck**

> Here is my current deck. Here is the new word list or chapter. Find what I am missing and give me only the new cards, in the same columns. Do not repeat existing cards.

**Audit a batch**

> Check these cards against the source. Flag anything unsupported, duplicated, unnatural, or uncertain. Do not rewrite them yet.

**Fix the style**

> These explanations are too long. Rewrite them as one short sentence each. Keep the same pattern and the same examples.

## The bottom line

Vibe coding taught us that you can build software without typing every line yourself. Vibe-decking is the same deal for language learning: you can build a serious deck without writing every card yourself.

But the lesson from vibe coding applies here too. You can generate without coding, and you can generate without writing cards — you just cannot skip reading the result. The AI is a fast, tireless assistant who has read a lot about your language and might still get a translation or a pronunciation wrong. Your job is to be the editor.

Used that way, it is not a gimmick. It is how I went from "I should make a proper Korean deck" to 327 grammar and reading cards — plus two cleaned-up vocabulary sets — without spending a week at the keyboard.
