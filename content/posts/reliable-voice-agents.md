---
title: "10 Principles for Building Reliable Voice Agents"
date: 2026-07-29T00:45:00+05:30
lastmod: 2026-09-28T12:00:00+05:30
draft: false
aliases:
  - /posts/livekit-voice-agents/
tags: ["AI", "Voice AI", "Reliability", "Engineering", "Agentic AI", "LiveKit"]
categories: ["Voice Agents"]
coauthors:
  - name: Kushagra Sahni
    url: https://www.linkedin.com/in/kushagra-sahni-171758201/
  - name: Vishal Shetty
    url: https://www.linkedin.com/in/vishal-shetty27/
  - name: Harshil Sethi
    url: https://www.linkedin.com/in/harshil-sethi-aa983221b/
  - name: Dibyashakti Moharana
    url: https://www.linkedin.com/in/dibyashakti-moharana/
cover:
  image: "/images/livekit-principles-banner.png"
  animated: "/images/livekit-principles-banner.svg"
  alt: "A chain of five connected dots; the middle one is hollow and an arc routes around it"
  caption: "Reliability for voice agents: when one piece fails, the call routes around it and keeps going"
description: "Ten hard-earned principles for keeping voice agents reliable in production, from seven months of incidents."
showToc: false
TocOpen: false
---

Voice agent demos are easy to build. A worker, an STT → LLM → TTS pipeline, and you have something that talks. The hard part is making it reliable.

We run AI call agents in production (ours are on LiveKit, but nothing below depends on that). A call lasts up to an hour. A lost recording is a compliance problem. A lost transcript means the user redoes the call. A silent agent means a human sitting in an empty room, wondering if anyone is there.

Over seven months of production traffic we fixed dozens of reliability issues. This post is what we learned, condensed into ten principles. For each: what it means, why we believe it, and how to apply it.

The principles are written to be read without touching code. If you build on LiveKit and want the exact APIs and error names behind each one, they are in the [code appendix](#appendix-livekit-code-references).

If you want the anatomy of the pipeline itself before the failure modes, read [Anatomy of an Agentic Voice Pipeline](/posts/agentic-voice-pipelines/) first.

---

## 1. Assume your shutdown code will never run

**The principle.** Don't rely on graceful shutdown to save anything. Save state continuously during the call, so a replacement process can pick up where the dead one left off.

![Snapshots throughout the call survive a process kill; the shutdown save never runs, and a replacement process resumes from the last snapshot](/images/reliability-p1.svg)

**Why.** The natural design is to do all post-call work (stop the recording, upload the transcript, notify your backend) in a shutdown callback. That callback can simply not run. Workers get killed by node preemption, host freezes, deploy drains, and the framework's own watchdog. We also hit a framework bug where an escaped exception skipped every registered callback. When a fresh process replaced a dead one, it started from zero: it re-greeted the user mid-call, and once uploaded an empty transcript over the real one.

**How to apply it.** Keep all session state in one in-memory store. Snapshot it to durable storage on a throttle; ours writes to S3 at most every 30 seconds. Trigger the save from every state write, so silent activity like a user typing gets saved too. Advance the throttle clock only when the upload succeeds, so failed uploads retry on the next write for free. On startup, look for a snapshot. If one exists, load it, skip the intro, and greet with "welcome back, let's continue". Test the recovery path, not just the save path.

## 2. Treat shutdown as a feature: register early, give it time

**The principle.** You can't rely on shutdown, but most calls end gracefully. Make that path solid: register cleanup before the session starts, and give it far more time than you think it needs.

![A callback registered after the greeting misses users who leave early; registered before the session, it covers the call from second zero](/images/reliability-p2.svg)

**Why.** Three ways we lost data on calls that ended normally. We registered the shutdown callback after the session start and the 40-second greeting, so a user who left during the greeting was never saved. The default shutdown timeout was far too short once cleanup did real work, and deploys killed our processes mid-cleanup. And our cleanup once crashed on a serialization error caused by state mutated by reference 40 minutes earlier: quiet corruption, detonating at the worst moment.

**How to apply it.** Register the shutdown callback **before** starting the session, even for resources that don't exist yet: declare them empty and let the callback check. Set the timeout for your worst case, not your average; ours grew to 20 minutes, one production incident at a time. Run independent cleanup steps in parallel. Make the state store return copies on read, so nothing mutates stored state by accident. Log three breadcrumbs (registered, invoked, completed), so a data-loss report tells you which failure mode you hit.

## 3. Make operations idempotent before you retry them

**The principle.** Retries are only safe on operations that can run twice without damage. Add the idempotency check first, then the retry.

![A retry without an idempotency check starts two recordings that clobber the same file; a check inside each attempt reuses the running one](/images/reliability-p3.svg)

**Why.** When our worker was respawned mid-call, the new process started a second recording of the same room. Both wrote to the same path, and the shorter one silently overwrote the longer one. A retry around that operation wouldn't have fixed anything. It would have caused the same bug more often.

**How to apply it.** Before starting a recording, check whether one is already running and reuse it. Put that check *inside* each retry attempt: if attempt one succeeded but the response got lost, attempt two finds it. Key every output path with a unique token, not just the room name; one platform incident recreated a room with the same name and its recording clobbered the original. When stopping a recording that already finished, expect the error, catch it, and fetch the final info anyway.

## 4. Give every dependency a fallback - a lazy one

**The principle.** Every external service (STT, TTS, LLM, avatar) will fail at some point. Have a plan B for each, and make it cost nothing until it's needed.

![Each pipeline stage has a dormant backup created only on failure; the avatar's backup is plain voice](/images/reliability-p4.svg)

**Why.** A single STT provider outage takes down every call. An LLM stream occasionally finishes without usable output; for us that surfaced as a confusing "sorry, I didn't catch that", and after enough repeats the call was force-ended for "technical difficulties" the user didn't cause.

**How to apply it.** Wrap STT in a fallback adapter, primary and backup chosen *per language*: the best English provider is not the best Spanish one. For the LLM, build a fallback that is only created when the primary fails: same prompts, same schema, different model. If that fails too, play a short localized apology and count it against an exception budget that eventually ends the call cleanly. Optional features like avatars get one retry, then drop to plain voice. Keep default strings for everything spoken. Count every fallback activation, so you notice when plan B is quietly carrying your traffic.

## 5. Degrade, never go silent

**The principle.** Whatever goes wrong, the user should keep hearing something. Silence is the worst failure mode a voice agent has.

![A staircase of graceful degradation from avatar and voice down to a clean goodbye; the only forbidden state is silence](/images/reliability-p5.svg)

**Why.** Every truly bad incident we had ended the same way: a human talking to a dead line. A wrong "call is ending" classification set an irreversible flag that dropped every later turn. Our avatar vendor's worker died mid-call and agent audio kept flowing to the dead avatar, so the user heard nothing; we now detect the disconnect, try one respawn, and otherwise route audio back to a direct track so the voice continues without the face. And a framework upgrade started auto-resuming speech after false interruptions while our old workaround *also* regenerated the reply, so the agent spoke twice. Re-audit your workarounds on every SDK upgrade.

**How to apply it.** Avoid one-way latches in conversation state. Make every error path speak: apology, nudge, or fallback answer. And give yourself a way to trigger the failure on demand; we added a debug button that force-kills the avatar, because a recovery path you can't test is a recovery path that doesn't work.

## 6. Flush at meaningful moments; throttle everywhere else

**The principle.** Periodic saving protects steady-state work. But some user actions make data unreachable afterward. Those moments must save immediately, bypassing the throttle.

![A timeline of throttled saves with force-save flags at every point of no return: next question, section end, call end](/images/reliability-p6.svg)

**Why.** We flushed users' written inputs to storage every 30 seconds. Sounds safe. But when a user moved to the next question, the frontend stopped sending updates for the old one. If the agent crashed just after the switch, the last few seconds on the old question were gone forever, and the automated evaluation graded stale work.

**How to apply it.** Save on events, gated by a throttle; no background timers. Add a force-save that skips the throttle, and call it at every point of no return: next question, section end, call end. Periodic saves are history; canonical finals are written at graceful shutdown, and if the process dies the snapshots reconstruct everything. Watch the small contracts too: our recovery silently failed for weeks over a float-vs-int timestamp in file names. File-naming conventions between services are API contracts.

## 7. Tune the voice pipeline for your conversation - and prove it with evals

**The principle.** VAD thresholds, noise cancellation, and turn-taking settings are not defaults you inherit. They depend on your users, languages, and the kind of conversation you're having. Tune them against real recordings, and keep those recordings as a test suite.

![Every audio knob sits between two failure modes, and the tuning set is every recording that ever misbehaved](/images/reliability-p7.svg)

**Why.** Our agent kept interrupting users mid-sentence. The cause was *endpointing*, tuned for snappy chat at a tenth of a second, while people answering real questions pause for whole seconds. Arabic-speaking users, who pause longer, got cut off constantly until dynamic endpointing adapted the delay per speaker. Background noise registered as speech and stopped the agent mid-sentence until interruptions were classified first. Aggressive noise cancellation deleted real speech; the gentler model let noise reach STT, which hallucinated words out of silence. As LiveKit put it while helping us debug: "we are oscillating between two failure modes." Change one stage of the audio pipeline and you re-open the tuning of its neighbors. And single-syllable answers ("sí", ~0.25s) were never detected by our first VAD, stalling calls at the consent question.

**How to apply it.** Collect every misbehaving recording as an eval case and re-run the suite whenever you touch a knob. When debugging audio, diff the raw recording against what the agent heard: the difference is exactly what your noise cancellation removed. Treat silence as a signal to classify, not a timeout. No audio ever received means a mic problem, so say that. Audio present means escalate gently: stay quiet, then "still there?", then a mic check. Your users are not in studios. They call from moving vehicles and kitchens with a kid crying in the background.

## 8. Never block the event loop

**The principle.** Your agent process is an async program supervised by a heartbeat. Any synchronous call in the pipeline can freeze the loop, and a frozen loop gets your process killed and the job re-dispatched.

![Heartbeat pings answered, then a 33-second synchronous call freezes the loop, pings go unanswered, and the process is killed](/images/reliability-p8.svg)

**Why.** Our framework pings each job process every 2.5 seconds and kills it after about 60 seconds without a response. We traced one wave of mid-call agent deaths to a synchronous 33-second RAG call buried inside an async tool. Another came from a blocking call in the entrypoint. From the user's side, both looked identical: the agent just vanished.

**How to apply it.** Every stage must be truly async; one blocking call inside an async function still freezes the loop. Don't poll with sleeps in streaming consumers; block on the queue and let the OS wake you. Attach an exception-logging callback to every fire-and-forget task. Unawaited task errors otherwise surface only at garbage collection, which can be never. That one change turned an unreproducible bug into a log line.

## 9. Know your platform's failure modes - and plan deploys around them

**The principle.** Whatever runs your workers (LiveKit, Pipecat on your own infra, anything) is a distributed system with its own failure modes. Some "agent bugs" are platform events, and some are self-inflicted through configuration.

![One symptom, the agent dropped mid-call, fans out to eight different root causes](/images/reliability-p9.svg)

**Why.** For months our top bug report was "the agent dropped mid-call." It had at least eight root causes, and misattributing one for another cost weeks each time. The most instructive:

- **Deploy drains.** New deployments drain old workers, by default for 30 minutes. Any longer call at deploy time was killed mid-call and logged as a *user-initiated* disconnect, which sent us hunting a frontend bug that didn't exist. We raised the drain window past our longest session and deploy off-hours.
- **Platform incidents.** A room-sweeper bug on the platform marked live rooms as ended. Node preemptions took out workers mid-job. From inside the agent, these look exactly like your own crashes.
- **Scale is a vendor-wide property.** Peak concurrency has to clear every vendor in the chain: your servers, the platform's session limits, and each STT and TTS provider's caps. The lowest limit is your real capacity. Autoscaling won't save you live either; warm capacity defaults are small and a new instance takes up to a minute. Before big events (we run ~2,000 calls a day) we pre-warm in the users' region and stagger session starts.
- **Dev workers steal prod jobs.** A laptop in dev mode with the same agent name competes for production dispatch. Different API keys did not isolate it. Use separate projects for dev and prod.
- **Your own backend counts.** One "platform failure spike" was our boot-data API timing out. Fetch boot data concurrently with connecting, and fail fast on missing fields.
- **The user's network counts too.** Corporate VPNs silently block the UDP paths WebRTC needs, and mic problems surface at the first question. Before the session, run a real dry run over the exact path the call will use: publish live mic and camera tracks, stream a few seconds, and check that packets actually leave the device. Problems surface on the setup screen with a retry button, not mid-call.

**How to apply it.** Keep the drain period longer than your longest session, deploy off-hours, pre-warm and stagger big events, isolate dev from prod, pre-check the user's connection, and monitor your own dependencies as part of the agent's SLA.

## 10. Measure everything, and attribute before you fix

**The principle.** You can't fix what you can't see, and you'll fix the wrong thing if you can't tell whose fault it is.

![In-memory counters ride the post-call webhook into a dashboard that alarms on degraded calls, not raw exceptions](/images/reliability-p10.svg)

**Why.** Two humbling examples. Our reliability counters were emitted in the sync code path, but production ran the async path, so the dashboard looked "healthy" while empty for months. And we spent a week auditing frontend code for user-initiated hangups that were actually deploy drains.

**How to apply it.** Count every fallible edge: recording start and stop, uploads, webhooks, fallback activations, recovery attempts. Agents are short-lived, so skip the live metrics pipe; accumulate counters in memory and attach them to the post-call webhook. That only works if shutdown works, which is why principles 1 and 2 come first. Alarm on *degraded calls*, not raw exception counts; ours fires at three recovered errors in a call. The agent and the recorder are participants in the same room, so they double as a control group: if they're stable while the human's connection flaps, the problem is on the user's side. And escalate quickly. Hosted traces expire in about 30 days, and twice we lost an investigation because the evidence aged out.

---

## Closing

None of this is exotic. It's ordinary distributed-systems discipline (durability, idempotency, fallbacks, observability) applied to a medium with an unusual failure mode. When a web service fails, the user sees an error page. When a voice agent fails, a human sits in silence, wondering if anyone is there. Build for that.

---

## Appendix: LiveKit code references

Our production stack is LiveKit. The concrete APIs, settings, and error names behind the principles, for readers implementing there.

**Principle 2 - shutdown.** Register `ctx.add_shutdown_callback(...)` before `session.start()`. For resources created later, declare `egress_id = None` up front and check `if egress_id:` inside the callback. Raise `WorkerOptions(shutdown_process_timeout=...)` - ours grew 60s → 120s → 600s → 1200s. Parallelize independent cleanup steps with `asyncio.gather`.

**Principle 3 - idempotent recording.** Call `list_egress(room_name=..., active=True)` before starting egress and reuse a running one, inside every retry attempt. `stop_egress` on a finished egress throws `failed_precondition`; catch it and fetch the final egress info anyway - the reliable recording start time is in `file_results[0].started_at` (nanoseconds).

**Principle 4 - fallbacks.** Wrap STT providers in LiveKit's `FallbackAdapter`. Guard `session.say()` so it never receives an empty string.

**Principle 5 - false interruptions.** On agents framework ≥1.5, false interruptions auto-resume - check the event's `resumed` flag before regenerating a reply, or the agent speaks twice.

**Principle 7 - turn-taking and audio.** Endpointing: `endpointing={"mode": "dynamic", "min_delay": 1.0, "max_delay": 6.0}` - `max_delay` is the biggest lever. Interruptions: `interruption={"mode": "adaptive"}` classifies audio before treating it as barge-in. Noise cancellation: BVC is the aggressive model, NC the gentler one - switching between them requires retuning the VAD threshold, and the egress recording is pre-NC audio while the agent hears post-NC, so diff them when debugging.

**Principle 8 - event loop.** The supervisor pings each job process every 2.5s and kills it after ~60s without a response. No sync calls inside `async def`; block on `queue.get(timeout=...)` instead of polling with `time.sleep`. Attach exception-logging done-callbacks to every `asyncio.create_task`.

**Principle 9 - platform.** Deploy drains default to 30 minutes and log the kill as `USER_INITIATED`. Workers started with `dev` mode and the same agent name compete for production dispatch even with different API keys - isolate by project. For the connection pre-check, publish real microphone and camera tracks and verify via `getStats()` that outbound media packet counts are increasing.

---

*Kudos to the team at Eightfold - the engineers who debugged every one of these incidents, often live with a user on the line, and turned each one into a fix, a metric, or a principle in this post. Seven months of production hardening is a team sport.*

## References

- [Anatomy of an Agentic Voice Pipeline](/posts/agentic-voice-pipelines/) - the companion post on how the pipeline itself works
- [LiveKit Agents documentation](https://docs.livekit.io/agents/) and [LiveKit Agents on GitHub](https://github.com/livekit/agents) - the framework behind the code appendix
- [Pipecat](https://github.com/pipecat-ai/pipecat) - an open-source alternative the same principles apply to
- [LiveKit WebRTC docs](https://docs.livekit.io/home/) - rooms, tracks, and connection internals behind principles 8 and 9
