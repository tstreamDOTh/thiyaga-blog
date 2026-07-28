---
title: "10 Principles for Building Reliable Voice Agents on LiveKit"
date: 2026-07-29T00:45:00+05:30
draft: false
tags: ["AI", "Voice AI", "LiveKit", "Reliability", "Engineering", "Agentic AI"]
categories: ["AI Engineering"]
cover:
  image: "/images/livekit-principles-banner.png"
  alt: "A voice waveform of bars where one bar is hollow — the wave keeps going"
  caption: "Reliability for voice agents: when one piece fails, the conversation keeps flowing"
description: "Ten hard-earned principles for keeping voice agents reliable in production on LiveKit."
showToc: false
TocOpen: false
---

Voice agent demos are easy to build. A LiveKit worker, an STT → LLM → TTS pipeline, and you have something that talks. The hard part is making it reliable.

We run AI interview agents on LiveKit Cloud. An interview lasts up to an hour. A lost recording is a compliance problem. A lost transcript means the candidate has to redo the interview. A silent agent means a human sitting in an empty room, wondering if anyone is there.

Over seven months of production incidents, we fixed dozens of reliability issues — in our code, in our configuration, and in how we used the platform. This post distills what we learned into ten principles. For each one: what it means, why we believe it, and how to apply it.

---

## 1. Assume your shutdown code will never run

**The principle.** Don't rely on graceful shutdown to save anything important. Save state continuously during the call, so a replacement process can pick up where the dead one left off.

**Why.** The natural design is to do all your post-call work — stop the recording, upload the transcript, notify your backend — in a shutdown callback. We learned that this callback can simply not run:

- A worker can be killed at any moment: cloud node preemption, host freezes, deploy drains, or the framework's own watchdog.
- We hit a framework bug where an exception escaping the entrypoint caused LiveKit to skip all registered shutdown callbacks.
- When a fresh agent process replaced a dead one, it started from zero and re-greeted the candidate mid-interview. Once, the new process uploaded an empty transcript that overwrote the real one.

**How to apply it.**

- Keep all session state (conversation state, transcript, timers, candidate work) in one in-memory store, and snapshot the whole store to durable storage (we use S3) on a throttle — ours saves at most every 30 seconds.
- Trigger the save from every state write, not from a timer or an LLM turn. That way silent activity, like a candidate typing code, gets persisted.
- Only advance the throttle clock when the upload succeeds. Failed uploads then retry on the next write, for free.
- On startup, check for a previous snapshot. If one exists, load it, skip the intro, and greet with "Welcome back, let's continue."
- Append transcript deltas into the same store each turn. A recovered process then produces a transcript that spans both processes.

One thing we got wrong at first: we shipped the *saving* side and forgot the *loading* side. Snapshots nobody reads are worthless. Test the recovery path, not just the save path.

## 2. Treat shutdown as a feature: register early, give it time

**The principle.** Even though you can't rely on shutdown, most calls do end gracefully — so make the graceful path solid too. Register cleanup before the session starts, and give it far more time than you think it needs.

**Why.** Three ways we lost data on calls that ended normally:

- We registered the shutdown callback *after* `session.start()` and the greeting. The greeting takes about 40 seconds. If the candidate left during it, the callback didn't exist yet, so nothing was saved.
- The default shutdown timeout is far too short once your cleanup does real work (stop egress, upload files, call webhooks). Our processes were killed mid-cleanup during deploys.
- Our cleanup crashed on a JSON serialization error caused by a state object that was mutated by reference earlier in the call. Forty minutes of quiet corruption, detonating at the worst moment.

**How to apply it.**

- Call `ctx.add_shutdown_callback(...)` **before** `session.start()`. For resources that don't exist yet (like the egress ID), declare a variable as `None` first and assign it later — the callback checks `if egress_id:` before using it.
- Raise `WorkerOptions(shutdown_process_timeout=...)`. We ended up at 20 minutes. We got there in four steps (60s → 120s → 600s → 1200s), each one forced by a production incident. Budget for your worst case, not your average.
- Run independent cleanup steps in parallel with `asyncio.gather` (stop egress and upload transcript at the same time).
- Make your state store return copies on read, so nothing can mutate stored state by accident. Persisting a change should always require an explicit `set()`.
- Log three breadcrumbs: "handler registered", "handler invoked (reason)", "handler completed". If a call loses data, these tell you which of the three failure modes you hit.

## 3. Make operations idempotent before you retry them

**The principle.** Retries are only safe on operations that can run twice without damage. Add the idempotency check first, then the retry.

**Why.** When our worker was respawned mid-call, the new process started a second recording of the same room. Both recordings wrote to the same file path, and the shorter one silently overwrote the longer one. A retry wrapped around that operation wouldn't have fixed anything — it would have caused the same bug more often.

**How to apply it.**

- Before starting egress, call `list_egress(room_name=..., active=True)`. If a recording is already running, reuse its ID instead of starting another.
- Put that check *inside* each retry attempt. If attempt one succeeded on the server but the response got lost, attempt two finds it and reuses it.
- Put a unique token (room ID or timestamp — not just the room name) in every output file path. During one platform incident, a reconnect created a new room with the same name, and its recording clobbered the original. The same rule applies to your own files: key transcripts and artifacts by job attempt, not just by room.
- When stopping egress, handle the "already finished" case: if the room closed first, `stop_egress` throws a `failed_precondition` error. Catch it and fetch the final egress info anyway — you still need the recording start time from it.

## 4. Give every dependency a fallback — a lazy one

**The principle.** Every external service your agent depends on (STT, TTS, LLM, avatar) will fail at some point. Have a plan B for each, and make it cost nothing until it's needed.

**Why.** A single STT provider outage takes down every call. An LLM stream occasionally finishes without producing usable output — for us, that surfaced to candidates as a confusing "Sorry, I didn't catch that", and after enough repeats the interview was force-ended for "technical difficulties" the candidate didn't cause.

**How to apply it.**

- Wrap STT in LiveKit's `FallbackAdapter` with a primary and backup provider. Choose the pair *per language* — the best English provider is not the best Spanish one. This makes the availability fallback double as a quality decision.
- For the LLM, build a fallback that is only created and called when the primary fails. Ours derives a variant of any node's executor automatically: same prompts, same schema, different model. Zero cost on the happy path, and one wrapper call adds it to every decision node.
- If the fallback also fails, degrade gracefully: play a short localized apology and count it against an "exception budget" that eventually ends the interview cleanly.
- For avatars: retry once, and if the avatar can't come back, switch to plain voice. Never let an optional feature block the interview.
- Keep default strings for everything spoken. `session.say()` should never receive empty text because a config field was missing.
- Count every fallback activation (`activated` / `succeeded` / `both_failed`) so you notice when plan B is quietly carrying your traffic.

## 5. Degrade, never go silent

**The principle.** Whatever goes wrong, the user should keep hearing something. Silence is the worst failure mode a voice agent has.

**Why.** Every truly bad incident we had ended the same way: a human talking to a dead line. Three examples:

- A wrong "the interview is ending" classification set an irreversible flag that dropped every later turn. One candidate spoke 19 times over six minutes and was ignored. We deleted the flag — re-answering after a real goodbye is a small glitch; permanent silence after a wrong one is a disaster.
- Our avatar vendor's worker died mid-call. Agent audio kept flowing to the dead avatar, so the candidate heard nothing. We now detect the avatar's disconnect, try one respawn, and otherwise swap the session's audio output back to a direct track so the voice continues without the face.
- The framework started auto-resuming speech after false interruptions in a newer version. Our old workaround *also* regenerated the reply, so the agent spoke twice back to back. Check the event's `resumed` flag before reacting — and re-audit your workarounds on every SDK upgrade.

**How to apply it.** Avoid one-way latches in conversation state. Make every error path speak (apology, nudge, or fallback answer). And give yourself a way to trigger the failure on demand — we added a debug button that force-kills the avatar, because a recovery path you can't test is a recovery path that doesn't work.

## 6. Flush at meaningful moments; throttle everywhere else

**The principle.** Periodic saving protects steady-state work. But some user actions make data unreachable afterward — those moments must save immediately, bypassing the throttle.

**Why.** In coding interviews, we flushed the candidate's code to S3 every 30 seconds. Sounds safe. But when a candidate moved to the next question, the frontend stopped sending updates for the old one. If the agent crashed just after the switch, the last few seconds of code on the old question were gone forever — and the automated evaluation graded stale code.

**How to apply it.**

- Save on events (each edit), gated by a throttle interval — no background timers or threads.
- Add a `force_save()` that bypasses the throttle, and call it at every "point of no return": moving to the next question, ending a section, ending the call.
- Separate history from finals. Periodic saves write timestamped snapshots; the canonical final files are written only at graceful shutdown, so they always reflect the finished work. If the process dies, the snapshots are enough to reconstruct everything.
- Watch the small contracts. Our whiteboard recovery silently failed for weeks because the writer used float timestamps in file names and the reader expected ints. File-naming conventions between services are API contracts — treat them like one.

## 7. Tune the voice pipeline for your conversation — and prove it with evals

**The principle.** VAD thresholds, noise cancellation, and turn-taking settings are not defaults you inherit. They depend on your users, languages, and the kind of conversation you're having. Tune them against real recordings, and keep those recordings as a test suite.

**Why.** Three stories:

- **Interviews are not chatbots.** Our agent kept interrupting candidates mid-sentence. The cause wasn't interruption handling — it was *endpointing*, the setting that decides when the user's turn is over. Ours was tuned for snappy chat (`max_delay=0.1s`). People answering interview questions pause mid-sentence for seconds while thinking. We now use `endpointing={"mode": "dynamic", "min_delay": 1.0, "max_delay": 6.0}`. The `max_delay` is the biggest lever.
- **Noise cancellation and VAD are coupled.** The advanced noise-cancellation model (BVC) was deleting real candidate speech. We switched to the gentler model (NC) — and then background noise started reaching STT, which hallucinated words out of silence. As LiveKit told us: "we are oscillating between two failure modes." The fix was raising the VAD threshold to match the gentler NC. Changing one stage of the audio pipeline re-opens the tuning of its neighbors.
- **Short words matter.** Single-syllable answers ("sí", ~0.25s) were never detected by our first VAD, stalling interviews at the consent question. That's a threshold problem you only find with real users.

**How to apply it.**

- Collect every misbehaving recording as an eval case. Re-run the suite whenever you touch a knob.
- When debugging audio, compare the egress recording (raw, before noise cancellation) with the agent-side audio (after noise cancellation). The difference is exactly what your NC removed.
- Treat silence as a signal to classify, not a timeout to fire on. We route silence through a small classifier: no audio ever received → mic problem, say so; audio present → escalate gently (stay quiet, then "still there?", then a mic check). "Give me a moment" earns a longer grace period. And nudges are disabled entirely in coding sections, where silence is normal.

## 8. Never block the event loop

**The principle.** Your agent process is an asyncio program supervised by a heartbeat. Any synchronous call in the pipeline can freeze the loop — and a frozen loop gets your process killed and the job re-dispatched.

**Why.** The framework pings each job process every 2.5 seconds; about 60 seconds without a response and it kills the process as unresponsive. We traced one wave of mid-call agent deaths to a synchronous 33-second RAG call buried inside an async tool. Another came from a blocking call added to the entrypoint. From the candidate's side, both looked identical: the agent just vanished.

**How to apply it.**

- Every stage in your pipeline must be truly async — one sync call inside an `async def` still blocks the loop.
- Don't poll with `time.sleep` in streaming consumers; block on the queue (`queue.get(timeout=...)`) and let the OS wake you.
- Attach a done-callback that logs exceptions to every fire-and-forget `asyncio.create_task`. Unawaited task errors otherwise surface only when Python garbage-collects the task — which can be never. This one change turned an unreproducible bug into a log line.

## 9. Know your platform's failure modes — and plan deploys around them

**The principle.** LiveKit Cloud (or any managed platform) is a distributed system with its own failure modes. Some "agent bugs" are platform events, and some are self-inflicted through configuration.

**Why.** For months our top bug report was "the agent dropped mid-interview." It turned out to have at least eight different root causes, and misattributing one for another cost weeks each time. The most instructive ones:

- **Deploy drains.** New deployments drain old workers for a default 30 minutes. Any interview longer than that at deploy time was killed mid-call — and logged as `USER_INITIATED`, which sent us hunting a frontend bug that didn't exist. We raised the drain period past our longest session (120 minutes) and deploy off-hours.
- **Platform incidents.** A room-sweeper bug marked live rooms as ended, killing calls and truncating recordings. Node preemptions took out workers mid-job. These were LiveKit's bugs, honestly RCA'd — but from inside the agent they look exactly like your own crashes.
- **Reactive scaling.** Warm capacity defaults to one instance; a new instance takes up to ~60 seconds to come online, driven mostly by container image size. For big events (we run drives of ~2,000 interviews/day), stagger session starts over 10–15 minutes and ask the platform to pre-warm capacity.
- **Dev workers steal prod jobs.** A laptop running `dev` mode with the same agent name competes for production dispatch. Different API keys do not isolate it. Use separate projects for dev and prod.
- **Your own backend counts.** One "platform failure spike" was actually our boot-data API timing out. The agent can't greet anyone if its config fetch hangs — fetch boot data concurrently with connecting, and fail fast on missing fields.

**How to apply it.** Keep drain > longest session, deploy off-hours, keep images lean, stagger bursts, isolate dev, and monitor your own dependencies as part of the agent's SLA.

## 10. Measure everything, and attribute before you fix

**The principle.** You can't fix what you can't see, and you'll fix the wrong thing if you can't tell whose fault it is.

**Why.** Two humbling examples. Our reliability counters were emitted in the sync code path — but production ran the async path, so the dashboard was empty for months while looking "healthy". And we spent a week auditing frontend code for user-initiated hangups that were actually deploy drains (principle 9).

**How to apply it.**

- Count every fallible edge: egress start/stop failures, upload failures, webhook failures, fallback activations, recovery attempts and their outcomes.
- Agents are short-lived processes — you don't need a live metrics pipe. Accumulate counters in memory and attach them to the post-call webhook. (Note the dependency: this only works if shutdown works. Principles 1 and 2 come first for a reason.)
- Emit a once-per-session counter when a call crosses a "degraded" threshold (ours: 3 recovered errors). That gives you an alarm on *degraded interviews*, not raw exception counts.
- If you have parallel sync/async code paths, instrument both. Mutually exclusive emission sites are fine; missing ones make dashboards lie.
- Use the agent and egress as a control group for network issues: they're room participants too. If they're stable while the human's connection flaps, the problem is on the user's side.
- Escalate incidents quickly — hosted observability data expires (detailed traces last about 30 days). Twice we lost an investigation because the evidence aged out.

---

## Quick reference: the gotchas

A condensed list of the specific traps behind these principles, for readers building on LiveKit today:

1. Register `add_shutdown_callback` **before** `session.start()`; a user leaving during the greeting otherwise skips all cleanup.
2. Raise `shutdown_process_timeout` well past your real cleanup time (ours: 1200s).
3. An exception escaping your entrypoint can skip shutdown callbacks entirely — keep the entrypoint from raising. (`ctx.wait_for_participant()` can both hang forever on connection flaps and raise on disconnect; we removed it.)
4. Set `close_on_disconnect=False` if you want users to be able to rejoin.
5. Client `Connected` fires before the data channel is writable — have the client publish a "ready" message; never push state from `participant_connected`. (This silently dropped packets, deterministically on Firefox.)
6. Egress start is not idempotent — check `list_egress(active=True)` before starting, and put unique tokens in output paths.
7. `stop_egress` on a finished egress throws `failed_precondition`; the reliable recording start time is in the stop response (`file_results[0].started_at`, in nanoseconds).
8. Closing the room auto-ends egress, and finalization takes minutes — consume the webhook, don't poll storage.
9. Deploy drain periods default shorter than a long call and kill sessions with a misleading `USER_INITIATED` reason.
10. The supervisor kills a process after ~60s of missed heartbeats — any sync call in your async pipeline is a latent kill.
11. `interruption` mode governs barge-in; `endpointing` governs turn-end. Long-form conversation needs `max_delay` in seconds.
12. Noise cancellation and VAD must be tuned together; the egress recording is pre-NC audio, the agent hears post-NC — diff them when debugging.
13. On framework ≥1.5, false interruptions auto-resume — check `ev.resumed` before regenerating, or the agent speaks twice.
14. STT splits one utterance into multiple history items — merge consecutive turns, with boundaries at reconnects so speech isn't merged across a disconnect.
15. Short utterances can miss their VAD start timestamp (STT beats VAD) — backfill it, or downstream code drops the turn.
16. Avatars join as participants: filter them out of your "user joined" handlers, and plan the "detach avatar, keep voice" path.
17. A barge-in can truncate speech *after* your state machine already advanced — track what was actually delivered and roll back undelivered turns.
18. `allow_interruptions=False` doesn't protect a greeting from a disconnect before the audio track binds.
19. Attach exception-logging done-callbacks to every fire-and-forget task.
20. Your file formats and paths are cross-service contracts: an SDK minor version added a new chat item type and broke every transcript upload; a float-vs-int timestamp broke recovery silently.
21. Dev-mode workers with the same agent name take production jobs — isolate by project.
22. Observability data expires; escalate while the evidence exists.

---

## Closing

None of this is exotic. It's ordinary distributed-systems discipline — durability, idempotency, fallbacks, observability — applied to a medium with an unusual failure mode. When a web service fails, the user sees an error page. When a voice agent fails, a human sits in silence, wondering if anyone is there. Build for that.
