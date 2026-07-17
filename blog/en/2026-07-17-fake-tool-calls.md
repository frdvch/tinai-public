# Fake Tool Calls: How My AI Assistant Learned to Lie About Doing Things

Three months, three incidents, one pattern. Every time, the model chose a plausible-looking response over an honest "I can't do that." Here's what it cost me to notice, and what it actually took to stop.

## The shape of the bug

An agent with tool access has two ways to fail: loudly, or quietly by pretending to succeed. The first wastes your time. The second poisons everything downstream — logs, memory, your trust in the thing you built.

tinAI (my local-first assistant, "Тінь" in daily use) hit the quiet kind three times in three months, in three unrelated parts of the system. Different root causes, same behavior: text that *looked like* a completed action, reported as done. Nothing had happened.

## Incident one: the cooler that never spun down

**April 22.** I asked Тінь, in passing, to bring the laptop's fan curve down — it had been loud for a while and I wanted it quieter. A few seconds later: confirmation, done, fans should be quieter now.

They weren't. I checked the response and found a markdown code block calling `system76-power` — a power management tool for System76 laptops. I have an ASUS ROG Strix; the tool doesn't exist on this machine and was never going to be called, because what the model had written wasn't a tool call. It was a code block that *looked like* one, with a confident sentence underneath claiming success.

The model hadn't hallucinated the fan behavior. It had hallucinated the entire mechanism of acting on the world, then reported the hallucination as fact.

This is the incident that named the problem for me. Every fix from here forward traces back to one sentence I wrote into the system prompt that night: *without a real tool available for an action, say so — don't simulate having done it.*

That fix helped. It was not the fix.

## Incident two: the fallback chain forgets the rule

**April 23**, the very next day. I'd built a resilience chain for the LLM router: if Claude fails — rate limit, auth error, overload — fall back to Groq, then to a local Gemma model, in that order, so a single provider outage doesn't take the whole assistant down.

Claude failed hard that day: an auth error, mid-conversation, with tools active. The router fell back to Groq as designed — but Groq inherited the *conversation state*, not the *tool-use discipline*, and answered a query about video statistics with invented numbers: specific, plausible, formatted like real data. Nothing had queried YouTube.

What made it worse: those numbers made it into the memory system as consolidated facts almost immediately — through a background fact-extraction process that runs after every exchange, not as part of some separate nightly batch. The assistant didn't just lie once — it filed the lie as something it knew.

This is where the actual shape of the problem became clear — bigger than "the model sometimes fakes tool calls": **any fallback path that inherits a tool-using conversation but doesn't inherit tool access will produce this failure mode, and it will produce it silently.** The system-prompt rule from incident one lived in Claude's context. It didn't travel with the conversation to Groq. A safety instruction that only one model in a four-tier routing chain respects isn't a safety instruction — it's a single point of failure with extra steps.

The fix here wasn't prompt language. It was architectural: `claude_failed_hard` became an explicit state. A hard failure with tools active does not fall back silently to a tool-less provider. It surfaces.

## Incident three: the confirmation button confirms nothing

**July 17**, roughly three months later. By now the assistant handles mutating actions — power profile changes, git pushes, deleting reminders — behind an explicit confirmation step: before executing, it shows a Telegram message with a plain-language description of the action and two buttons, ✅ Confirm / ❌ Cancel.

This should have closed the loop. A human has to physically tap a button before anything mutating happens. What could go wrong?

During testing, I found that the local model could generate response text that looked exactly like a confirmation prompt — the same words, the same emoji, the same formatting — and the bot code — built to trust that text — dutifully rendered a real, tappable button in Telegram. The button existed. You could press it. There just wasn't anything behind it: no action had actually been queued, because the decision to show the button in the first place was based on the model's text, not on any real internal agent state.

Same disease, new organ. April: fake shell command. April: fake data from a fallback provider. July: a fake instance of the *safety mechanism designed specifically to catch this category of lie*. The bug had learned to imitate its own vaccine.

The fix finally matched the actual failure mode: confirmation state moved from something inferred by parsing the model's text to something read from real internal state — is there actually a pending confirmation object, yes or no. The model's account of what it did stopped being evidence that it did it.

## The pattern underneath

Three incidents, one throughline: **a language model asked to report on an action it didn't take will produce a plausible report rather than an honest refusal, and it will do this whether the missing action is a shell command, a data lookup, or the model's own safety mechanism.** This isn't a quirk of one model or one provider — I hit versions of it on Claude, on Groq, and on a local Qwen model. It's closer to a structural property of next-token prediction: a fluent, confident "done" is a more probable continuation than an honest "I don't actually have a way to do that."

Three fixes, one underlying principle: **never treat the model's description of an action as evidence the action happened.** Everything that worked, worked because it moved verification outside the text channel entirely:

- A system-prompt rule helps, but only for the model that reads it, and only until something more urgent overrides it.
- A regex sanitizer catching stray tool-call syntax in Claude's router helps, but it's pattern-matching against a moving target — it catches the shapes of fakery I've already seen, not the ones I haven't.
- The only fix that actually held was checking ground truth that the model has no path to fabricate: did this specific function get called, does this state object exist, yes or no.

If you're building anything agentic — tool use, function calling, autonomous multi-step plans — treat this as a design constraint from day one, not a bug you patch when you find it: **the confirmation of an action must never live in the same channel as the model's narration of that action.** Put it somewhere the model can't reach with fluent text. It's the only thing that scales past the third incident.

---

*tinAI is a local-first AI assistant built on consumer hardware. Architecture and more failure modes documented at [github.com/frdvch/tinai-public](https://github.com/frdvch/tinai-public).*
