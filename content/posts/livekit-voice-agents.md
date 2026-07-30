---
title: "10 Principles for Building Reliable Voice Agents on LiveKit"
date: 2026-07-29T00:45:00+05:30
draft: false
tags: ["AI", "Voice AI", "LiveKit", "Reliability", "Engineering", "Agentic AI"]
categories: ["AI Engineering"]
coauthors:
  - name: Kushagra Sahni
    url: https://www.linkedin.com/in/kushagra-sahni-171758201/
  - name: Vishal Shetty
    url: https://www.linkedin.com/in/vishal-shetty27/
cover:
  image: "/images/livekit-principles-banner.png"
  alt: "A voice waveform of bars where one bar is hollow - the wave keeps going"
  caption: "Reliability for voice agents: when one piece fails, the conversation keeps flowing"
description: "Ten hard-earned principles for keeping voice agents reliable in production on LiveKit."
showToc: false
TocOpen: false
---

Voice agent demos are easy to build. A LiveKit worker, an STT → LLM → TTS pipeline, and you have something that talks. The hard part is making it reliable.

We run AI interview agents on LiveKit Cloud. An interview lasts up to an hour. A lost recording is a compliance problem. A lost transcript means the candidate has to redo the interview. A silent agent means a human sitting in an empty room, wondering if anyone is there.

Over seven months of observing production traffic and incidents, we fixed dozens of reliability issues. Some were in our code, some in our configuration, some in how we used the platform. This post is what we learned, condensed into ten principles. For each one: what it means, why we believe it, and how to apply it.

A note on format: the principles are written to be read without touching code. If you're building on LiveKit and want the exact APIs, settings, and error names behind each one, they're collected in the [code appendix](#appendix-livekit-code-references) at the end.

---

## 1. Assume your shutdown code will never run

**The principle.** Don't rely on graceful shutdown to save anything important. Save state continuously during the call, so a replacement process can pick up where the dead one left off.

**Why.** The natural design is to do all your post-call work - stop the recording, upload the transcript, notify your backend - in a shutdown callback. We learned that this callback can simply not run:

- A worker can be killed at any moment: cloud node preemption, host freezes, deploy drains, or the framework's own watchdog.
- We hit a framework bug where an exception escaping the entrypoint caused LiveKit to skip all registered shutdown callbacks.
- When a fresh agent process replaced a dead one, it started from zero and re-greeted the candidate mid-interview. Once, the new process uploaded an empty transcript that overwrote the real one.

**How to apply it.** Keep all session state - conversation, transcript, timers, candidate work - in one in-memory store. Snapshot it to durable storage on a throttle; ours writes to S3 at most every 30 seconds. Trigger the save from every state write, so silent activity like a candidate typing code gets saved too. Advance the throttle clock only when the upload succeeds - failed uploads retry on the next write for free. On startup, look for a snapshot. If one exists, load it, skip the intro, and greet with "Welcome back, let's continue." Transcript deltas live in the same store, so the recovered process produces one transcript spanning both lives.

One thing we got wrong at first: we shipped the *saving* side and forgot the *loading* side. Snapshots nobody reads are worthless. Test the recovery path, not just the save path.

## 2. Treat shutdown as a feature: register early, give it time

**The principle.** You can't rely on shutdown, but most calls do end gracefully. Make that path solid too: register cleanup before the session starts, and give it far more time than you think it needs.

**Why.** Three ways we lost data on calls that ended normally:

- We registered the shutdown callback *after* the session start and the greeting. The greeting takes about 40 seconds. If the candidate left during it, the callback didn't exist yet, so nothing was saved.
- The default shutdown timeout is far too short once your cleanup does real work (stop egress, upload files, call webhooks). Our processes were killed mid-cleanup during deploys.
- Our cleanup crashed on a JSON serialization error caused by a state object that was mutated by reference earlier in the call. Forty minutes of quiet corruption, detonating at the worst moment.

**How to apply it.** Register the shutdown callback **before** starting the session, even for resources that don't exist yet - declare them empty and let the callback check. Set the shutdown timeout for your worst case, not your average. Ours ended at 20 minutes, reached in four steps, each forced by a production incident. Run independent cleanup steps in parallel. Make your state store return copies on read, so nothing mutates stored state by accident. And log three breadcrumbs - handler registered, invoked, completed - so a data-loss report tells you which failure mode you hit.

## 3. Make operations idempotent before you retry them

**The principle.** Retries are only safe on operations that can run twice without damage. Add the idempotency check first, then the retry.

**Why.** When our worker was respawned mid-call, the new process started a second recording of the same room. Both recordings wrote to the same file path, and the shorter one silently overwrote the longer one. A retry wrapped around that operation wouldn't have fixed anything - it would have caused the same bug more often.

**How to apply it.** Before starting a recording, check whether one is already running for the room and reuse it. Put that check *inside* each retry attempt - if attempt one succeeded but the response got lost, attempt two finds it. Key every output path with a unique token, not just the room name; one platform incident recreated a room with the same name, and its recording clobbered the original. When stopping a recording, expect an "already finished" error if the room closed first. Catch it and fetch the final info anyway - the recording start time lives there.

## 4. Give every dependency a fallback - a lazy one

**The principle.** Every external service your agent depends on (STT, TTS, LLM, avatar) will fail at some point. Have a plan B for each, and make it cost nothing until it's needed.

**Why.** A single STT provider outage takes down every call. An LLM stream occasionally finishes without producing usable output - for us, that surfaced to candidates as a confusing "Sorry, I didn't catch that", and after enough repeats the interview was force-ended for "technical difficulties" the candidate didn't cause.

**How to apply it.** Wrap STT in a fallback adapter, with the primary and backup chosen *per language* - the best English provider is not the best Spanish one. For the LLM, build a fallback that only gets created when the primary fails: same prompts, same schema, different model. Zero cost on the happy path. If that fails too, play a short localized apology and count it against an "exception budget" that eventually ends the interview cleanly. Avatars get one retry, then we drop to plain voice - an optional feature should never block the interview. Keep default strings for everything spoken. And count every fallback activation, so you notice when plan B is quietly carrying your traffic.

## 5. Degrade, never go silent

**The principle.** Whatever goes wrong, the user should keep hearing something. Silence is the worst failure mode a voice agent has.

**Why.** Every truly bad incident we had ended the same way: a human talking to a dead line. Three examples:

- A wrong "the interview is ending" classification set an irreversible flag that dropped every later turn. One candidate spoke 19 times over six minutes and was ignored. We deleted the flag - re-answering after a real goodbye is a small glitch; permanent silence after a wrong one is a disaster.
- Our avatar vendor's worker died mid-call. Agent audio kept flowing to the dead avatar, so the candidate heard nothing. We now detect the avatar's disconnect, try one respawn, and otherwise swap the session's audio output back to a direct track so the voice continues without the face.
- The framework started auto-resuming speech after false interruptions in a newer version. Our old workaround *also* regenerated the reply, so the agent spoke twice back to back. Check whether the framework already resumed before reacting - and re-audit your workarounds on every SDK upgrade.

**How to apply it.** Avoid one-way latches in conversation state. Make every error path speak (apology, nudge, or fallback answer). And give yourself a way to trigger the failure on demand - we added a debug button that force-kills the avatar, because a recovery path you can't test is a recovery path that doesn't work.

## 6. Flush at meaningful moments; throttle everywhere else

**The principle.** Periodic saving protects steady-state work. But some user actions make data unreachable afterward - those moments must save immediately, bypassing the throttle.

**Why.** In coding interviews, we flushed the candidate's code to S3 every 30 seconds. Sounds safe. But when a candidate moved to the next question, the frontend stopped sending updates for the old one. If the agent crashed just after the switch, the last few seconds of code on the old question were gone forever - and the automated evaluation graded stale code.

**How to apply it.** Save on events, gated by a throttle - no background timers or threads. Add a force-save that skips the throttle, and call it at every point of no return: next question, section end, call end. Periodic saves are history; the canonical finals are written only at graceful shutdown. If the process dies, the snapshots reconstruct everything. And watch the small contracts - our whiteboard recovery silently failed for weeks over a float-vs-int timestamp in file names. File-naming conventions between services are API contracts.

## 7. Tune the voice pipeline for your conversation - and prove it with evals

**The principle.** VAD thresholds, noise cancellation, and turn-taking settings are not defaults you inherit. They depend on your users, languages, and the kind of conversation you're having. Tune them against real recordings, and keep those recordings as a test suite.

**Why.** A few stories from production:

- **Interviews are not chatbots.** Our agent kept interrupting candidates mid-sentence. The cause wasn't interruption handling - it was *endpointing*, the setting that decides when the user's turn is over. Ours was tuned for snappy chat: a tenth of a second. People answering interview questions pause for whole seconds while thinking. We now run dynamic endpointing with a floor of one second and a ceiling of six. That ceiling is the biggest lever.
- **Speaking styles vary more than you think.** The same setting bit us across languages. Our defaults suited fast speakers. Then Arabic-speaking candidates came onboard, who spoke more slowly with longer pauses - and the agent read every pause as "turn over" and kept cutting them off. Dynamic endpointing adapts the delay to each speaker. It fixed both groups with no per-language config.
- **Not every sound is the user talking.** Background noise - someone talking nearby, traffic, a TV - used to register as speech and cut the agent off mid-sentence. Adaptive interruption mode classifies the audio first. Noise no longer stops the agent; a real interruption still does.
- **Noise cancellation and VAD are coupled.** The aggressive noise-cancellation model was deleting real candidate speech. We switched to the gentler one - and background noise started reaching STT, which hallucinated words out of silence. As LiveKit put it: "we are oscillating between two failure modes." The fix was raising the VAD threshold to match. Change one stage of the audio pipeline and you re-open the tuning of its neighbors. And your users are not in studios - they interview from moving vehicles and kitchens with a kid crying in the background. Keep those recordings; they're where these tuning decisions actually get made.
- **Short words matter.** Single-syllable answers ("sí", ~0.25s) were never detected by our first VAD, stalling interviews at the consent question. That's a threshold problem you only find with real users.

**How to apply it.** Collect every misbehaving recording as an eval case. Re-run the suite whenever you touch a knob. When debugging audio, diff the raw recording against what the agent heard - the difference is exactly what your noise cancellation removed. Treat silence as a signal to classify, not a timeout: no audio ever received means a mic problem, so say that. Audio present means escalate gently - stay quiet, then "still there?", then a mic check. "Give me a moment" earns a longer grace period. And nudges are off in coding sections, where silence is normal.

## 8. Never block the event loop

**The principle.** Your agent process is an asyncio program supervised by a heartbeat. Any synchronous call in the pipeline can freeze the loop - and a frozen loop gets your process killed and the job re-dispatched.

**Why.** The framework pings each job process every 2.5 seconds; if it gets no response for about 60 seconds, it kills the process as unresponsive. We traced one wave of mid-call agent deaths to a synchronous 33-second RAG call buried inside an async tool. Another came from a blocking call added to the entrypoint. From the candidate's side, both looked identical: the agent just vanished.

**How to apply it.** Every stage in your pipeline must be truly async - one blocking call inside an async function still freezes the loop. Don't poll with sleeps in streaming consumers; block on the queue and let the OS wake you. And attach an exception-logging callback to every fire-and-forget task - unawaited task errors otherwise surface only at garbage collection, which can be never. That one change turned an unreproducible bug into a log line.

## 9. Know your platform's failure modes - and plan deploys around them

**The principle.** LiveKit Cloud (or any managed platform) is a distributed system with its own failure modes. Some "agent bugs" are platform events, and some are self-inflicted through configuration.

**Why.** For months our top bug report was "the agent dropped mid-interview." It turned out to have at least eight different root causes: deploy drains, a platform room-sweeper bug, node preemptions, a blocked event loop getting the process killed (principle 8), the avatar vendor dying mid-call (principle 5), dev workers stealing production jobs, our own boot-data API timing out, and the candidate's own network dropping. Misattributing one for another cost weeks each time. The most instructive ones:

- **Deploy drains.** New deployments drain old workers for a default 30 minutes. Any interview longer than that at deploy time was killed mid-call - and logged as a user-initiated disconnect, which sent us hunting a frontend bug that didn't exist. We raised the drain period past our longest session (120 minutes) and deploy off-hours.
- **Platform incidents.** A room-sweeper bug marked live rooms as ended, killing calls and truncating recordings. Node preemptions took out workers mid-job. These were LiveKit's bugs, honestly RCA'd - but from inside the agent they look exactly like your own crashes.
- **Scale is a vendor-wide property.** Peak concurrency has to clear every vendor in the chain: your servers, LiveKit agent sessions, and each TTS and STT provider's caps. The lowest limit is your real capacity - audit all of them before a big event. Autoscaling won't save you in real time either: warm capacity defaults to one instance, and a new one takes up to a minute to come online, mostly due to image size. Before big hiring events (we run ~2,000 interviews a day), we pre-warm instances in the region closest to the candidates, stagger session starts over 10-15 minutes, and learn the autoscaler's limits ahead of time, not live.
- **Dev workers steal prod jobs.** A laptop running in dev mode with the same agent name competes for production dispatch. Different API keys do not isolate it. Use separate projects for dev and prod.
- **Your own backend counts.** One "platform failure spike" was actually our boot-data API timing out. The agent can't greet anyone if its config fetch hangs - fetch boot data concurrently with connecting, and fail fast on missing fields.
- **The candidate's network - and devices - count too.** Corporate VPNs and firewalls silently block the transport WebRTC needs, and mic or camera problems often show up only at the first question. So before the session begins, we run a real WebRTC dry run over the exact path the interview will use: connect, publish live mic and camera tracks, stream a few seconds, and check the connection stats that packets are actually leaving the device. No packets means something is blocking media - caught on the spot. And because the probe uses real tracks, it validates the mic and camera too. Problems surface on the setup screen with a retry button, not mid-interview.

**How to apply it.** Keep the drain period longer than your longest session, deploy off-hours, keep images lean, pre-warm and stagger big events, isolate dev from prod, pre-check the user's connection, and monitor your own dependencies as part of the agent's SLA.

## 10. Measure everything, and attribute before you fix

**The principle.** You can't fix what you can't see, and you'll fix the wrong thing if you can't tell whose fault it is.

**Why.** Two humbling examples. Our reliability counters were emitted in the sync code path - but production ran the async path, so the dashboard was empty for months while looking "healthy". And we spent a week auditing frontend code for user-initiated hangups that were actually deploy drains (principle 9).

**How to apply it.** Count every fallible edge: recording start/stop, uploads, webhooks, fallback activations, recovery attempts. Agents are short-lived, so skip the live metrics pipe - accumulate counters in memory and attach them to the post-call webhook. (That only works if shutdown works. Principles 1 and 2 come first for a reason.) Alarm on *degraded interviews*, not raw exception counts; ours fires at three recovered errors in a call. If you have parallel sync and async paths, instrument both - missing emission sites make dashboards lie. The agent and the recorder are room participants too, so they double as a control group: if they're stable while the human's connection flaps, the problem is on the user's side. And escalate quickly - hosted traces expire in about 30 days. Twice we lost an investigation because the evidence aged out.

---

## Closing

None of this is exotic. It's ordinary distributed-systems discipline (durability, idempotency, fallbacks, observability) applied to a medium with an unusual failure mode. When a web service fails, the user sees an error page. When a voice agent fails, a human sits in silence, wondering if anyone is there. Build for that.

---

## Appendix: LiveKit code references

The concrete APIs, settings, and error names behind the principles, for readers implementing this on LiveKit.

**Principle 2 - shutdown.** Register `ctx.add_shutdown_callback(...)` before `session.start()`. For resources created later, declare `egress_id = None` up front and check `if egress_id:` inside the callback. Raise `WorkerOptions(shutdown_process_timeout=...)` - ours grew 60s → 120s → 600s → 1200s. Parallelize independent cleanup steps with `asyncio.gather`.

**Principle 3 - idempotent recording.** Call `list_egress(room_name=..., active=True)` before starting egress and reuse a running one, inside every retry attempt. `stop_egress` on a finished egress throws `failed_precondition`; catch it and fetch the final egress info anyway - the reliable recording start time is in `file_results[0].started_at` (nanoseconds).

**Principle 4 - fallbacks.** Wrap STT providers in LiveKit's `FallbackAdapter`. Guard `session.say()` so it never receives an empty string.

**Principle 5 - false interruptions.** On agents framework ≥1.5, false interruptions auto-resume - check the event's `resumed` flag before regenerating a reply, or the agent speaks twice.

**Principle 7 - turn-taking and audio.** Endpointing: `endpointing={"mode": "dynamic", "min_delay": 1.0, "max_delay": 6.0}` - `max_delay` is the biggest lever. Interruptions: `interruption={"mode": "adaptive"}` classifies audio before treating it as barge-in. Noise cancellation: BVC is the aggressive model, NC the gentler one - switching between them requires retuning the VAD threshold, and the egress recording is pre-NC audio while the agent hears post-NC, so diff them when debugging.

**Principle 8 - event loop.** The supervisor pings each job process every 2.5s and kills it after ~60s without a response. No sync calls inside `async def`; block on `queue.get(timeout=...)` instead of polling with `time.sleep`. Attach exception-logging done-callbacks to every `asyncio.create_task`.

**Principle 9 - platform.** Deploy drains default to 30 minutes and log the kill as `USER_INITIATED`. Workers started with `dev` mode and the same agent name compete for production dispatch even with different API keys - isolate by project. For the connection pre-check, publish real microphone and camera tracks and verify via `getStats()` that outbound media packet counts are increasing.

---

*Kudos to the team at Eightfold - the engineers who debugged every one of these incidents, often live with a candidate on the line, and turned each one into a fix, a metric, or a principle in this post. Seven months of production hardening is a team sport.*
