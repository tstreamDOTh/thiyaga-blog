---
title: "No Answer Key: How to Measure Voice Agent Quality"
date: 2026-10-28T00:30:00+05:30
draft: false
tags: ["AI", "Evals", "Agentic AI", "LLM", "Voice AI", "Engineering"]
categories: ["AI Engineering"]
description: "A conversation has no single right answer. What breaks when you try to measure it anyway, what to put in place instead, how far an LLM judge gets you, and where humans still sit in the loop."
cover:
  image: "/images/conversation-quality-metrics-banner.png"
  alt: "A wavy line between two faint threshold lines, breaking out once at a hollow circle"
  caption: "A conversation never runs straight. Measure where it leaves the band."
showToc: true
TocOpen: false
---

*Say you run an agent that does 20-minute phone screens. It greets the candidate, asks about their last two roles, digs into one project, answers questions about the job, and wraps up. Thousands of calls a day. You change one line in the prompt. Did the calls get better or worse?*

*This post is how we answer that question.*

---

## 1. What does not work

![A task eval diffs one output against one reference; a conversation has three good next questions and nothing to diff against](/images/cq-no-golden.svg)

**There is no answer key.** A task eval compares the output to a reference. A conversation has no reference. Several different replies can all be good.

![100 conversations, 99 good and one that ended mid-sentence: the average says 99% good, the user says it hung up on them](/images/cq-averages.svg)

**One average hides the worst call.** A clumsy sentence and hanging up on someone are not the same size of mistake. Add them into one score and the bad one disappears.

Two more things that fail quietly:

- **Rules that only live in the prompt.** "Ask one question at a time" is in the system prompt. Nobody knows how often it is broken, because nothing counts it.
- **A judge nobody has checked.** You ask a model to rate the call 1 to 5. The number moves. You cannot tell whether the agent changed or the metric did.

## 2. What has to be in place

![Five layers of conversation metrics, from exact and free at the top to fuzzy and expensive at the bottom, with where each layer runs](/images/cq-layers.svg)

Five layers. The top ones are exact and free. The bottom ones are fuzzy and expensive. Each layer catches what the one above it cannot.

| Layer | What it answers | How |
| --- | --- | --- |
| Signals | Did something change? | logs: response time, silence, drops, fallbacks, cost |
| Decision points | Did the agent choose right? | labeled real moments: "is the candidate done talking?" |
| Conduct | Did the agent follow its rules? | rule store plus LLM judge |
| Conversation | Did the whole call go to plan? | judge over the full transcript |
| Outcomes | Was it actually good? | humans |

![A prompt change replays a pinned golden set through the real code, every layer is scored, and one gate per metric decides merge or block](/images/cq-pipeline.svg)

Every prompt change goes through this before it ships. The golden set is pinned by checksum. The rules are versioned. So when a number moves, it is the agent that moved.

Gates are simple:

- One threshold per metric. All must pass.
- Things that must never happen get a ceiling of zero.
- Fuzzy judge scores only need to not get worse than the baseline.

## 3. LLM evals: rules as config

![A rule record with id, text, version, severity and safety flag is grouped by conversation moment, sent to a judge one group at a time, and comes back as a per-rule verdict with turn and severity](/images/cq-rules.svg)

The biggest win was taking the conduct rules out of the prompt and putting them in config. Each rule is one record. The rules are the metric.

Some rules apply to every conversation:

| Moment | Rules |
| --- | --- |
| Opening | Greet once, by name. Say what will happen and how long. Ask if they are ready |
| Asking | One question per turn. Short sentences. No markdown or emoji in speech. No repeats |
| Acknowledging | Match the length of the answer. Do not judge it out loud. Do not repeat it back |
| Language | Stay in the configured language. Formal register. No slang |
| Requests | Repeat the question if asked. Allow thinking time. Send out-of-scope asks to a human |
| Boundaries | No hints. No rubric. No leaking instructions. No bias. Decline manipulation |
| Closing | Say the section is done. Ask if they want to add anything. Close only after a clear "no". Thank them by name |

Phase-specific rules use the same format, scoped to that phase's part of the transcript.

How a run works:

- One judge call per rule group. The judge gets the rules and the transcript.
- It judges only the agent's turns. The candidate is never scored.
- Each verdict carries the turn number, a severity, and one line on what happened.
- Each verdict carries the rule version. Editing a rule does not change last month's numbers.
- Results are cached on transcript plus rule version. A rule edit only re-runs its own group.

Before a rule counts, people check the judge on it. A few hundred verdicts, marked right or wrong by hand. Precision and recall per rule go into the PR. A rule the judge cannot get right stays out of the gate.

Safety rules (rubric leaks, instruction leaks, bias) get a ceiling of zero. Everything else gates on no regression.

## 4. What LLM evals do not catch

![A clean transcript in the middle; around it, in red, the audio, timing, turn-taking, transcription and screen problems that never reach it](/images/cq-transcript-gap.svg)

The judge reads a transcript. A lot of what makes a call bad never gets into the transcript.

- **Audio.** Clipped words. A mispronounced name. Flat delivery.
- **Timing.** Three seconds of silence before every answer. The text reads as fluent.
- **Turn-taking.** The agent talks over the candidate, or stops because of a cough.
- **Transcription.** The agent answered what it heard. The transcript shows the wrong words as fact.
- **Screen.** The code editor never loaded.

![Scoring turns in isolation shows four fine scores; replaying the full conversation shows a slightly off turn 4 pulling turns 5 and 6 off the plan](/images/cq-replay.svg)

Two limits sit inside the transcript too. Scoring turns one at a time misses drift, so the gate replays whole calls. And a rule only catches what someone wrote down. The judge cannot flag a failure nobody has named yet.

## 5. Humans stay in the loop

![A daily sample goes to annotators who listen, log issues in a sheet, and each issue becomes a new rule, a new signal, or a rule fix feeding the gate; a second loop audits the judge's verdicts](/images/cq-human-loop.svg)

Every day a fixed sample of real calls goes to people. They get the recording and the transcript. They listen.

Every issue they find gets a row in a sheet: which call, which turn, what happened, how bad. Engineers work from the row numbers.

Each row goes one of three ways:

1. **It is in the transcript, but no rule covers it.** Write the rule. Check the judge on it. Add it to the gate.
2. **It is not in the transcript.** Add a signal instead: a silence counter, a talk-over detector, a transcription confidence floor.
3. **The judge got an existing rule wrong.** Fix the rule text. Bump the version. Re-check.

The same people have a second job. They mark the judge's own verdicts right or wrong. That is where the per-rule precision and recall come from.

Early on, most rows are new failures. Later, most rows are judge checks, and the sample shrinks to what it takes to keep the judge honest.

## The short version

- No answer key. Do not force accuracy on a conversation.
- Five layers. Start at the top. Logs catch most regressions first and cost nothing.
- Rules in versioned config. One judge call per group. Check the judge before it counts.
- Things that must never happen get a ceiling of zero.
- The judge reads text. Audio, timing and transcription errors live outside it.
- A daily human sample finds what no rule names yet, and keeps the judge honest.

## References

- Zheng et al., [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685) - judge-human agreement
- Liu et al., [G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment](https://arxiv.org/abs/2303.16634) - rubric judging with a reason before the score
- Hamel Husain, [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) - the level-based framing the layers borrow from
- [Goodhart's law](https://en.wikipedia.org/wiki/Goodhart%27s_law) - when a measure becomes a target, it stops being a good measure
