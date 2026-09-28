---
title: "Anatomy of an Agentic Voice Pipeline"
date: 2026-09-28T11:30:00+05:30
draft: false
tags: ["AI", "Voice AI", "Agentic AI", "Engineering", "Architecture"]
categories: ["AI Engineering"]
cover:
  image: "/images/voice-pipeline-anatomy-banner.png"
  alt: "A line passing through three small circles on a pastel teal background"
  caption: "A voice pipeline: one signal, passing through stages"
description: "The stages of an agentic voice pipeline, what each one is for, where the latency goes, and when to skip the pipeline entirely."
showToc: false
TocOpen: false
---

The fastest way to give a product a voice today is a **speech-to-speech model**: one realtime model that hears audio and speaks audio directly, like OpenAI's Realtime API or Gemini Live. And for simple voice interaction it is genuinely good. Ask your app a question, dictate a note, chat hands-free while driving. One API call, natural prosody, low latency. When voice is a supplementary experience layered on top of a product, this is the right tool, and you can stop reading here.

This post is about the other case: when voice *is* the product. An agent that runs an hour-long call, follows a process, calls tools mid-conversation, serves ten languages, and gets audited when something goes wrong. For that, production systems use a **cascaded pipeline**: separate STT, LLM, and TTS stages with text flowing between them. It is more work, but the text at every seam is what makes the agent controllable, debuggable, and evaluable, and each stage can be the best available for the job instead of whatever one vendor bundled.

My [last voice post](/posts/reliable-voice-agents/) was about keeping such agents alive in production. This one explains the machine itself: the stages, why each exists, where the time goes, and a closer look at the architecture fork above. Nothing here is specific to one vendor. The same anatomy applies to Pipecat, LiveKit Agents, or a pipeline you build by hand.

## The shape of the machine

A voice agent is not a request-response service. It is a loop that never stops running. Audio streams in from the user, flows through a chain of stages, and audio streams back out. Every stage runs concurrently, all the time.

![The agentic voice pipeline: audio flows from the user through transport, VAD, turn detection, and STT into the agent core, then back out through TTS](/images/voice-pipeline-map.svg)

Seven pieces do the work. The rest of this post walks through them in the order audio meets them.

You do not have to build this from scratch, and mostly you should not. [Pipecat](https://github.com/pipecat-ai/pipecat) and [LiveKit Agents](https://github.com/livekit/agents) are open-source frameworks that implement the whole loop; you plug in providers and write the agent core. [LiveKit Cloud](https://cloud.livekit.io/), [Daily](https://www.daily.co/), and similar platforms run the transport and worker infrastructure as a managed service. Understanding the anatomy still matters. Every knob these frameworks expose maps to a stage below, and when a call misbehaves, you debug it stage by stage, whoever runs the stages.

## Transport: WebRTC

Audio reaches your agent over WebRTC. It is worth knowing what it is actually doing for you, because when calls go bad, this layer is usually where the evidence is.

WebRTC ships audio over UDP and accepts that packets will be lost. Waiting for a retransmit would mean audible gaps, so it does something smarter: a jitter buffer reorders packets that arrive late and out of order, packet loss concealment synthesizes the missing few milliseconds so the ear never notices, and the codec adapts bitrate as the network degrades. On the capture side you get echo cancellation, which is what stops the agent's own voice from coming back through the user's speakers and interrupting itself. Every one of these is a hard real-time DSP problem, solved and battle-tested for free.

Two things follow for the pipeline. First, the audio your agent hears is already processed: buffered, concealed, echo-cancelled. When you debug "the STT misheard the user", the raw network conditions are one layer below the audio you logged. WebRTC exposes per-connection stats (packet loss, jitter, round-trip time), and they belong in your metrics next to the STT confidence scores.

Second, the user's network is part of your pipeline. WebRTC needs UDP paths that corporate VPNs and firewalls silently block, and the fallback relays add latency you never budgeted. Probe the connection with real media before the conversation starts, not during it.

## VAD: is anyone talking?

Voice activity detection is the pipeline's gatekeeper. It answers one question, frame by frame: is this audio speech or not? Everything downstream keys off it. STT only transcribes when VAD says someone is talking. Interruption handling fires when VAD detects speech while the agent has the floor.

The knob that matters is the sensitivity threshold. Too sensitive and a TV in the background becomes "the user is speaking", which cuts the agent off mid-sentence. Too strict and a quiet "yes" never registers, and the call stalls. There is no universal setting. It depends on your users' environments, and it is coupled to your noise cancellation: clean up the audio more aggressively and the same threshold behaves differently. Tune them together, against real recordings.

## Turn detection: are they done talking?

This is the hardest problem in the pipeline, and the most underrated. VAD tells you the user went quiet. It cannot tell you whether they finished their thought or paused to think. Humans resolve this with grammar, intonation, and context. A pipeline has to guess.

The naive approach is a fixed silence timeout. Short timeouts feel snappy in demos and interrupt real users constantly, because people pause for whole seconds mid-answer. Long timeouts stop the interruptions and make every reply feel sluggish. The better approach is semantic turn detection: a small model looks at the transcript so far and estimates whether the sentence is complete, then stretches or shrinks the silence window to match. "What I meant was..." earns a long wait. "No." does not.

![Turn-taking as a state machine: listening, thinking, and speaking, with failure transitions for false endpoints, barge-in, and false interruptions](/images/voice-turn-states.svg)

The state machine view makes the failure modes concrete. A false endpoint fires the transition too early and the agent talks over the user. A missed barge-in ignores a real interruption and the agent steamrolls on. A false interruption lets background noise pause the agent mid-answer. Every tuning decision in a voice pipeline is a trade between these three arrows.

Barge-in deserves one more sentence. When a real interruption lands, stopping the audio is the easy part. You also have to cancel the in-flight LLM call, flush the TTS buffer, and roll conversation state back to the words the user actually heard, not the words the agent generated. Skip that last step and the agent believes it said things the user never received.

## STT: audio to text

Speech-to-text is the most commodity stage, but two properties matter for the pipeline. First, it must stream. A streaming STT emits interim transcripts while the user is still talking, so by the time the turn ends, most of the text is already transcribed and the finalize step costs around a hundred milliseconds instead of a full re-pass. Second, quality is per-language. The best English provider is rarely the best Tamil or Arabic one, so the STT stage should be a routing decision, not a constant.

## The agent core: where it becomes agentic

Everything up to here is plumbing. The core is what makes the pipeline agentic instead of a voice frontend for a chatbot: an LLM plus tools plus persistent state.

Tools change the shape of the latency problem. A text agent that takes five seconds to look something up shows a spinner. A voice agent that takes five seconds produces five seconds of dead air, and dead air reads as a dropped call. Two patterns help. Speak before you act: "let me pull that up" costs one cheap generation and buys the tool call time. And run slow tools async where possible: kick off the lookup, keep the conversation going, weave the result in when it lands.

State matters more than in text agents because the process is fragile. Workers get killed mid-call. If conversation state lives only in process memory, a crash means the replacement re-greets a user who is forty minutes in. Snapshot state continuously so a fresh process can resume the conversation. The [previous post](/posts/reliable-voice-agents/) covers this failure mode in detail.

## TTS: text to audio

Same rule as STT: it must stream. The metric is time to first byte, because the user unblocks the moment the first chunk plays, not when the clip finishes. In practice you sentence-chunk: as soon as the LLM has produced one complete sentence, send it to TTS and start playing, while the rest of the reply is still generating. And every TTS stream must be cancellable mid-utterance, or barge-in cannot work.

## The latency budget

Now the numbers. Conversation has a rhythm. People leave gaps of a few hundred milliseconds between turns. Past roughly 800 ms of silence, the pause stops feeling like thinking and starts feeling like lag, and users repeat themselves, which makes everything worse.

![Latency waterfall: a naive sequential pipeline takes about 4.7 seconds to first audio, a streamed pipeline about 1.15 seconds](/images/voice-latency-waterfall.svg)

The waterfall shows why streaming is not an optimization but the design. Run the stages sequentially, each waiting for the previous to finish, and you are at several seconds before the user hears anything. Stream every stage and the same components respond in around a second. Two things to notice in the streamed bar:

- The endpointing wait is the single biggest slice, and it is pure policy, not compute. Semantic turn detection spends that budget where it is needed instead of paying it on every turn.
- The LLM's time-to-first-sentence matters. Its total generation time mostly does not, because TTS hides it.

When the budget still cannot be met, cheat honestly: acknowledgements ("mm-hmm", "got it") and filler phrases buy hundreds of milliseconds and make the agent feel more human, not less.

## Cascaded vs speech-to-speech, properly

Back to the fork from the top, now with the full picture. Everything above describes the cascaded pipeline: separate STT, LLM, and TTS with text at every seam. The speech-to-speech model collapses all of it into one model that hears audio and speaks audio directly.

![Cascaded pipeline vs speech-to-speech model: the cascade gives text at every seam and per-stage control, the realtime model gives lower latency and native prosody](/images/voice-cascaded-vs-s2s.svg)

Speech-to-speech wins on the things audio carries and text drops. It hears hesitation and tone. It speaks with natural prosody. It removes two model hops from the latency path.

The cascade wins on everything operational, and that is why production systems still mostly cascade. Text at every seam means you can log it, redact it, evaluate it, and audit it. Each stage swaps independently, so you can pick the best STT per language and fall back per stage when a vendor has a bad day. Tool calling is mature. Costs are legible. With a speech-to-speech model you get one vendor, one failure domain, and no text seam to inspect when a call goes wrong.

My read: speech-to-speech is where the medium is heading, and the gap closes every quarter. But if you are shipping to production today, especially anything regulated or multilingual, the cascade's control is worth its latency. A reasonable middle path exists too: cascade for the pipeline, and spend the saved complexity on semantic turn detection and streaming discipline, which is where most of the perceived quality lives anyway.

## Closing

The anatomy is the easy half. Every stage here will also fail in production: the STT vendor will have an outage, the process will die mid-call, the event loop will block. That half is the [previous post](/posts/reliable-voice-agents/). Read them as a pair: this one is how the machine works, that one is how to keep it alive.

## References

- [10 Principles for Building Reliable Voice Agents](/posts/reliable-voice-agents/) - the production-hardening companion to this post
- [Silero VAD](https://github.com/snakers4/silero-vad) - the open-source voice activity detector most pipelines start with
- [Pipecat](https://github.com/pipecat-ai/pipecat) and [LiveKit Agents](https://github.com/livekit/agents) - open-source frameworks that implement this anatomy
- [A Simplest Systematics for the Organization of Turn-Taking for Conversation (Sacks, Schegloff, Jefferson, 1974)](https://pure.mpg.de/rest/items/item_2376846_3/component/file_2376845/content) - the linguistics behind why endpointing is hard
