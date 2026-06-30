# Degrading instead of breaking: what local_only = true actually buys you

*2026-06-26*

---

*This post is part of a series on the technical decisions behind tinAI. It covers the 4-tier routing in detail elsewhere. This one is about what happens to that routing stack when the cloud tiers stop being an option.*

---

On June 9, the Gemini credits ran out. The shadow service went down and stayed down for two weeks.

That outage is the reason `local_only = true` exists now, and it's worth being specific about why a config flag is the actual fix here, not just a patch.

## What the outage exposed

tinAI's routing stack is local Qwen first, then Groq, then Claude Haiku, then Claude Sonnet, escalating by task complexity. The assumption baked into that design was that the cloud tiers would generally be available, and the local tier was the fallback for cost and latency, not the only option.

When the Gemini credits ran out, that assumption broke. The service didn't gracefully degrade to local-only. It just stopped, because the routing logic still expected the cloud tiers to be reachable and didn't have a clean path for "they're not, and that's fine, keep going anyway."

Two weeks of downtime is a long time to not have an assistant because of a billing detail unrelated to whether the core system actually works.

## The fix

`local_only = true` in `config/config.toml` removes Claude and Groq from the routing decision entirely. Every request, regardless of complexity, goes to the local Qwen model. No fallback logic tries to reach the cloud and fails. No retries against a tier that isn't going to answer.

This sounds like a small change but it's a different posture. Before, the cloud tiers were assumed-available infrastructure the system depended on. Now they're optional infrastructure the system can use when present and ignore cleanly when not. Re-enabling them later is one line and a restart.

Alongside this, voice input got disabled for now and text-only became the default through Qwen. The system runs noticeably leaner — fully offline, no external API dependency at all in this mode.

## The classifier decision that came out of the same pressure

A related fix from this stretch: simple commands — volume changes, exact numbers, max/min — now route through a regex classifier instead of going to the LLM at all. "Set volume to 60%" doesn't need a 4B parameter model to parse. It needs pattern matching that returns in milliseconds instead of seconds, and doesn't depend on Qwen's occasionally unreliable tool-calling.

This wasn't originally about cost. It came from watching Qwen fail to call tools cleanly on commands that didn't need an LLM's judgment at all. But it has the same shape as the local_only decision: identify what doesn't actually require the expensive path, and stop sending it down that path.

## The principle underneath both

A personal assistant that breaks when a third-party billing limit is hit is not actually reliable, no matter how good it is when everything's funded. The system should have a floor — a mode where it keeps working, even in a reduced way, independent of anyone else's infrastructure or anyone else's invoice.

`local_only` is that floor now. It's not the preferred mode. Cloud tiers add real capability for complex tasks. But the system no longer collapses when they're unavailable. It degrades, predictably, to something that still works.

---

*tinAI is a local-first AI assistant running on an ASUS ROG laptop with an RTX 3050. Architecture and decisions are documented at [github.com/frdvch/tinai-public](https://github.com/frdvch/tinai-public).*
