# QTradeX Docs — Tone Guide

## The premise

The docs should make the reader feel like they're inventing QTradeX themselves, one step at a time.

Not "here's how the framework works." Not "let me show you what I built." But: *"you have a problem, you try something, it almost works, you try the next thing — oh, that's why it's designed this way."*

Every feature should feel like an inevitability the reader just arrived at, not an explanation they're receiving.

## How 3Blue1Brown does it

Grant Sanderson doesn't explain math. He creates conditions where the viewer feels the next step before he says it. The formula lands *after* the intuition, as the natural encoding of something you already understand.

Key mechanisms:

**Start smaller than you think.** Not with the full problem — with the simplest concrete instance that still has the shape of the real thing. A single bot class with no methods. One indicator. The reader needs to see the skeleton before you add muscle.

**Let the reader feel the gap.** Before introducing a feature, first show what breaks without it. The reader should sense "wait, but what about...?" just before you answer. Example: autorange isn't "a method that computes warmup" — it's "your strategy would receive NaN indicators on day 1. That's a problem. Here's how the framework solves it."

**One concept per beat.** Each paragraph introduces exactly one new idea. No compound reveals. The reader's working memory is small — treat it that way.

**"Why" is the throughline.** Every feature description answers: what problem does this solve, and why is this the right shape for the solution? The API surface is the *conclusion*, not the subject.

**Name things when they earn it.** Jargon is a reward for understanding, not a prerequisite. Use the concrete description first; introduce the term afterward as a convenient label for something the reader already knows.

**The reader's questions drive the structure.** Read each section and ask: "what would I be wondering right now?" If the next paragraph doesn't answer that question, restructure.

## Voice

- Active voice. "You'll notice that..." not "it should be noted that..."
- Conversational but not cute. Think "smart colleague thinking through a problem aloud."
- Short sentences. Varied length, but default to short.
- No self-congratulation. "This elegant design..." or "powerful feature" — never. Let the reader conclude it's elegant.
- Contractions are fine. "You'll", "don't", "it's".

## Concrete patterns

| Instead of | Write |
|---|---|
| "X allows you to do Y" | "You need Y. Here's how X does it." |
| "X is a powerful feature for..." | *Delete the sentence. Show an example where X is the obvious solution.* |
| "The framework provides..." | *Start with what the reader wants to accomplish.* |
| "First, let me explain..." | *Start with the concrete thing the reader can see happen.* |

## What to avoid

- **Framing features as the author's achievements.** The reader doesn't care what you built. They care what they can do with it.
- **Reference-bloat.** Don't list every parameter upfront. Introduce options when they matter.
- **Explaining the obvious.** If the code is clearer than the prose, delete the prose.
- **Long paragraphs.** Three sentences max before a line break. The eye needs rest stops.
- **"We" as the discoverer.** "We can see that..." distances the reader. "You can see..." or just state the observation.
- **Burying the lede.** If the reader needs to know X before Y, put X first. If you find yourself saying "as we saw earlier," the ordering is probably wrong.

## AI writing tics to kill on sight

These patterns are the default output of every LLM. They signal "an AI wrote this" faster than anything. Stamp them out.

- **"Not X, but Y."** The rhetorical negation. "Not a framework, but a toolkit." "Not a function, but a philosophy." It's a cheap way to sound deep. Just say what it is.
- **"Something — the noun phrase."** "Backtesting — the art of simulating history." "Optimization — the search for better numbers." The em dash + appositive is the most recognizable LLM construction in existence. If you catch yourself doing it, restructure: make the noun phrase the sentence.
- **Lists of three.** Rhetorical triplets ("fast, flexible, and reliable") are an AI tic. One or two adjectives is honest. Three is incantation. Exception: when you're actually listing three concrete things, not three vibes.
- **Excessive em dashes.** They're fine — in measured doses. If a paragraph has more than one, you're not writing, you're generating. Use a period.
- **"The key to..."** It's never the key. Just state the relationship.
- **"In other words..."** Say it in the right words the first time.
- **"Think of it as..."** The reader decides what to think of it as. Show them the thing.
- **"Let's dive in."** No. You're already in.
- **"That said" / "However" / "Of course"** as paragraph starters.** When every paragraph hedges the last one, the structure is wrong, not the connection.

## Testing the tone

Read each paragraph aloud. Does it sound like someone explaining something they just figured out? Or like someone reading from a spec sheet?

If the paragraph reads the same after removing the code blocks, rewrite it. The prose should be doing work the code can't.
