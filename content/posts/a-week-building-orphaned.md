---
title: "A week building ORPHANED: letting Claude write the game, keeping it honest"
date: 2026-08-08
draft: false
tags: ["python", "claude", "ai-assisted-coding", "testing", "gamedev"]
---

ORPHANED is a tiny always-on-top window with a pixel creature living inside it. It grows on real-world time whether the program is open or not, it fights hostile "programs" when you feel like picking a fight, and if you neglect it long enough it dies — permanently — and gets written into an archive of every version that came before.

I spent about a week building it. But the part worth writing down isn't the game. It's this: I can't really write a program this size on my own, so I didn't. Claude wrote almost all of the code. My job was deciding what the thing should do, running it, playing it, and catching it when it quietly drifted off the rails. Two things kept a week of AI-generated code from collapsing into an unmaintainable mess — a written rulebook and a test suite — and I didn't appreciate either until I'd already made the mistakes that made me need them.

If you're a beginner about to build something bigger than you can hold in your head, this is the stuff I wish I'd known on day one.

## The first thing that tripped me: the files don't come with you

The creature started as a single Python file using tkinter for the window, and grew into five: data tables, the pet's logic, the combat, the window, and an entry point. Early on, though, I was re-uploading all of that into every new conversation, because I assumed I had to.

That's risky in a way I didn't see at first. When Claude doesn't have your current files, it can reconstruct an *older* version from an earlier chat — and you won't notice until something breaks in a way that makes no sense. For a project whose whole point is careful state (saving, loading, aging offline), editing from a half-remembered version is exactly the wrong foundation.

The fix was mundane and I wish someone had just said it to me: put the actual files into the Project's knowledge — added as Project files, not pasted into a message. After that, every conversation could read the current code directly instead of guessing. If you're going to work across many sessions — and you will, especially if you keep bumping into usage limits like I did — do this before anything else.

## The rulebook

Here's the thing I did backwards. I didn't start with rules. I started with a working-ish game that had already grown messy, and *then* had Claude write the rulebook: a "master prompt" that lives in the project and gets read before anything gets touched.

Writing it *after* the mess turned out to be the good part, because the rules could point at mistakes we'd actually made. The core of it is a one-way dependency chain:

```
content.py   data tables: enemies, moves, items, dialogue
model.py     the pet — bars, traits, xp, save/load, archive
combat.py    battles — turn resolution, rewards, costs
ui.py        the tkinter window
```

Nothing imports "upward," and the bottom three files never import tkinter at all. That one rule — the window holds no game rules, no number that decides an outcome — is what makes everything else possible. Because the game logic doesn't need a window, the whole thing can run headless, which means it can be *simulated* and *tested* without a human sitting there clicking.

The part of the rulebook I didn't expect to lean on was a list of bug classes we'd *already shipped once*, written down so we'd stop reshipping them. Things like "store time in seconds, never float hours" (hours had drifted off the growth thresholds after weeks of offline time) and "reset must never delete the archive" (it did, once). A rulebook made of your own scars is worth more than a generic best-practices list.

## The test suite, or: how I stopped saying "seems to work"

Before this, every check was "run it and look." For a game whose entire premise is that time passes while the program is *closed*, "run it and look" is nearly useless — I'd have to shut my laptop for two days to see the failure mode.

So we built a test suite from zero, in three tiers. The middle tier taught me the most: **invariant tests** — properties that must hold for *any* input, not just the case you happened to think of. The clearest one:

> Calling `advance(24 hours)` once must give exactly the same result as calling `advance(1 hour)` twenty-four times.

That reads like a technicality. It's actually the whole game. When you close the laptop on Friday and open it on Monday, the creature catches up on the missed time in one big jump — and that jump has to land in the same place it would have if you'd left it running the whole weekend. If the two paths disagree, offline aging can't be trusted, and offline aging *is* the game.

Tests like that immediately caught two bugs I'd never have found by playing:

- A **dead pet could still gain personality traits and refill its stats**, because the "are you alive?" check lived in the window instead of in the pet itself. Anything that talked to the pet directly walked straight around it.
- A **corrupt save could take a perfectly good archive down with it** — if the creature's data was garbled but the archive of past lives was intact, the whole load failed and the archive got dropped.

Both are the kind of thing that silently eats someone's save.

Running the suite is one command:

```bash
python3 -m unittest discover -s . -p "test_*.py"
```

And there are headless modes for eyeballing balance without opening a window:

```bash
python3 orphaned.py --simulate 168   # print the neglect curve over a week
python3 orphaned.py --arena          # print the combat win-rate matrix
```

The third test tier pins those numbers to *ranges* rather than exact values — a tier-1 enemy stays winnable by a young creature, a tier-4 enemy stays unwinnable — so if I tune a constant and accidentally wreck the difficulty curve, a test yells at me instead of a player discovering it three sessions later.

## The lesson I actually needed: "looks like a choice" isn't "is a choice"

My favourite bug wasn't a crash. The creature is supposed to develop a *personality* from how you care for it — doting, distant, playful, anxious. But in an early version, personality points were credited on *every* button press. Which meant the drain rates — how fast the bars emptied — decided the personality, not the player. One trait won literally every run, no matter how I played.

The system looked like a meaningful choice and was completely decorative. And "decorative thing pretending to be a choice" is a trap AI code falls into constantly, because it will happily build something that *looks* right.

The fix was two parts. First: only credit personality for *discretionary* care — doing something when the creature didn't strictly need it yet — capped at one point per hour so you can't farm it by mashing a button. Second, and this is the rule I now apply to everything: **prove the system branches.** There's a test that simulates several different play styles and checks that the personalities actually come out *different*. If "doting" and "distant" produce the same creature, the choice was a lie, and the test fails.

## Smaller things that bit me

**The test that "failed" but was right.** A statistical test on item-drop rates went red once. My first instinct was "the code is broken." It wasn't — my *assertion* was wrong. I'd guessed a certain item should drop under 15% of the time; the actual math worked out to exactly 20%. The fix was to correct the test, not the code. "The test failed" and "the code is broken" are not the same sentence, and confusing them sends you off fixing things that were never broken.

**The sci-fi quotes I couldn't have.** I wanted a "talk" button that pulled from a big file of sci-fi movie quotes. Claude wouldn't reproduce copyrighted quotes at any scale — the right call — so instead we wrote a big pool of *original* lines in the creature's own voice. Honestly better for the game; the whole point of the thing is that it isn't quoting anyone.

**The rename that left a landmine.** Partway through I renamed the game from AWAKENING to ORPHANED. What I didn't catch for a while: this left `awakening.py` and `orphaned.py` sitting side by side, byte-for-byte identical except one word, so a fix to one silently didn't apply to the other. There was also a save path that said `~/.orphaned` in the code but `~/.awakening` in a leftover comment. Renaming is never just the file you're looking at — grep for the old name *everywhere* before you call it done.

**Claude can't see the window.** This one's structural and worth knowing if you build any GUI this way. The sandbox Claude runs code in doesn't have tkinter, so it could never actually *open* the window it was writing. Every visual fix was reasoned from the math, not seen — which made the click-testing my job, on my machine, every time. The bug that proved it: encounter text overlapping the choice buttons so the bottom line was unreadable. It looked fine in the code and was fine in the math; it was only wrong once I played it. When the AI says "this should render correctly," treat "should" as load-bearing.

## Where it stands, honestly

It works. I've played it for real — the longest run was about seventeen hours. The `ui.py` file, which ballooned past 700 lines while every session bolted on one more fix, is finally split back down to around 400 across a few view modules. The suite passes on my machine.

What's *not* done, so you don't think I'm further along than I am:

- **It's not on GitHub yet.** It lives on my laptop. That's the next thing I'm eyeing.
- **No packaging.** Turning this into something you double-click — let alone something on Steam — is a separate mountain I've only looked at from the bottom.
- **One combat move is knowingly broken.** A "probe" move meant to reveal the enemy's next intent doesn't actually reach a point where you can act on it. Rather than quietly leaving it, it's tracked as a test that is *expected to fail* — a broken thing that announces itself every time the suite runs, so I can't forget it's there.

That last one is the whole post in miniature. The difference between a week of AI code that holds together and a week of AI code that rots isn't that the AI stops making mistakes. It's whether the mistakes are written down where they'll nag you — in a rulebook, in a failing test, in a list of bugs you've already shipped once — instead of being discovered by a player two weeks later when their save disappears.
