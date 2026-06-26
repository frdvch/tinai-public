# When Good Ideas Aren't Worth the Cost: Removing Night Mode

*2026-05-09*

---

Some features die not because they were wrong, but because the cost of keeping them alive exceeds what they give back. Night mode is that story.

## What it was supposed to do

The idea was simple: when you leave a large download or a long job running overnight, the system should quietly shift into a low-impact state on its own. Dim the screen. Silence notifications. Throttle background activity. Let the machine work without disturbing anything.

On paper, this made sense. tinAI already knows about system state — CPU load, active processes, battery, audio profile. Adding a night mode felt like a natural next step: one more thing the assistant handles so you don't have to think about it.

## What actually happened

The problem was the environment tinAI runs in.

A personal Linux machine at night is not a clean slate. There are package updates mid-download, PipeWire doing something unexpected with audio, background sync processes waking up, the GPU contending with Ollama. Night mode had to interact with all of it — and every interaction was a potential conflict.

The feature worked in isolation. It broke in context. And debugging it meant staying close to a machine that was supposed to be running unattended, across multiple evenings, chasing failures that only reproduced at 2 AM after a suspend cycle.

After a few rounds of fixes, the pattern was clear: each fix introduced a new edge case. The feature was not converging toward stable.

## The decision

On 2026-05-09, night mode was removed entirely. Only a stub remains — `session.night_mode()` — a leaf function that does nothing, kept so the call sites don't break if the concept ever comes back in a simpler form.

The screen dimming can be handled by system power settings. The silence is handled by the audio profile toggle already in the Telegram quick-menu. The things night mode was automating were either already solved elsewhere or not worth solving in code.

## The actual lesson

The feature was trying to coordinate too many things that it didn't own. Screen brightness, audio, background processes — these are the operating system's domain. Reaching across those boundaries from an AI assistant layer is possible, but the blast radius when something goes wrong is large and the failure modes are hard to reproduce on demand.

There is a version of this feature that could work: narrow scope, explicit trigger, no ambient state tracking. But that is a different feature, and it can wait until there is a concrete reason to build it.

For now, the night mode is gone. The machine runs cleaner without it.

---

*tinAI is a local-first AI assistant running on an ASUS ROG laptop with an RTX 3050. Architecture and decisions are documented at [github.com/frdvch/tinai-public](https://github.com/frdvch/tinai-public).*
