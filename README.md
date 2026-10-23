# tinAI — local AI partner, not a replacement

> Read this in [Українська](README.uk.md)

A local-first AI assistant built on consumer hardware (ASUS ROG laptop, RTX
3050 4 GB VRAM) with cost routing between local and cloud LLMs, persistent
memory with a real forgetting curve, and a transparent HUD. Built by a
Ukrainian war veteran — local-first as a response to seeing infrastructure
collapse during war.

The code is private. The architecture and decisions are documented in this
repository. Access on request. See [Source code access](#source-code-access)
below.

## Why

- **AI alongside humans, not replacing them.** Like a pilot and a navigator:
  one flies the plane, the other reads the maps and the conditions. The
  decision stays with the human. The AI carries the cognitive load.
- **Local-first.** Because I have seen what happens to infrastructure
  during war. Because privacy is not a comfort, it is a necessity. Because
  depending on OpenAI, Anthropic, or Google is the same shape of
  dependence as depending on Gazprom gas.
- **Consumer hardware as a deliberate constraint.** RTX 3050 4 GB VRAM. If
  it works here, it works almost anywhere.

## What it does

tinAI is a personal assistant that runs locally on Linux. Text through
Telegram is the working path: it classifies the request, routes it to the
cheapest acceptable model, calls tools through a local MCP layer, and
answers. It remembers across days through a memory system that actually
forgets old, unimportant things instead of accumulating forever.

The voice stack is built and present — Whisper STT for Ukrainian, Piper and
Edge TTS for output, double-clap activation — but it ships disabled
(`[voice] enabled = false`, `live_mode = false`). Voice was the original
premise of this project and it turned out to be the part I use least:
typing into Telegram from a phone beats talking at a laptop in nearly every
situation I actually have. The code stays, because the cost of keeping it is
one config flag.

Core modules:

- **HUD.** Frameless, transparent, always-on-top circular overlay. A PyQt6
  shell wrapping a QWebEngine view over a local static server, fed by a
  websocket. No chat input and no transcript panel — both were removed in
  the 2026 redesign; everything routes through the backend.
- **Memory.** ChromaDB plus a networkx knowledge graph plus an exponential
  forgetting curve. Nightly consolidation at 02:00.
- **Telegram.** Owner-gated bot, the primary interface. Text and voice (OGG)
  input, a persistent reply keyboard, and inline buttons for per-item
  actions. Mutating actions require an explicit confirm tap.
- **Obsidian.** Read and write, with a privacy window and a secret scanner
  gating the write paths — see
  [ARCHITECTURE.md](ARCHITECTURE.md#obsidian-integration).
- **Finance.** Read-only connector to a bank's personal API: account
  balances and statements, with spending categories derived from merchant
  type codes rather than from a model. Replaced a manual receipt tracker —
  see [the post on deleting it](blog/en/2026-10-23-deleting-the-feature-i-used-once.md).
- **Shift schedule.** A rotating work schedule kept in the vault, editable
  from Telegram, with a reminder before each shift and an acknowledge button.
- **System metrics.** One source of truth for temperatures, fans, and load
  (averages and peaks, named sensors), plus a persistent temperature log so
  "is it hotter than last week" is an answerable question.
- **24 MCP servers** for shell, git, notes, reminders, web search, vision,
  music, vault, bank statements, network, and windows. Large surface,
  acknowledged.
- **Voice (disabled).** Whisper for Ukrainian STT, Piper for offline TTS,
  double-clap activation. Present in the code, switched off in config.
- **Cost routing** across local Qwen 3 4B and Claude Haiku today, with Groq
  Llama and Claude Sonnet tiers still in the code but unreachable while
  `local_only = true`. A circuit breaker protects against local LLM latency
  spikes.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the detailed write-up.

Brief overview:

- **Cost routing, as designed:** four tiers — local Qwen 3 4B → Groq Llama →
  Claude Haiku → Claude Sonnet.
- **Cost routing, as it actually runs:** `local_only = true`, so the ladder
  collapses to two live tiers — local Qwen for short tool intents, Claude
  Haiku for everything else, with local as the fallback. The Groq and Sonnet
  branches are still in the router and currently unreachable. Documented as
  two separate facts because conflating them would be marketing.
- **Subscription transport:** the Haiku tier runs through the `claude -p`
  CLI on an OAuth subscription, not through the API wallet, with the API key
  deliberately stripped from the subprocess environment.
- **Memory 2.0:** ChromaDB facts/episodes/reflections plus a networkx
  knowledge graph plus an exponential forgetting curve (halflife 30 days,
  threshold 0.1). The decay is real, not planned.
- **Circular PyQt6/QWebEngine HUD** communicating over `ws://localhost:8765`,
  with an origin allowlist on the socket.
- **Obsidian integration** — read and write, with secret scanning and a
  privacy window gating writes, and two separate path resolvers after a
  path-traversal bug.
- **Telegram remote access** for text and voice, with confirm tokens on
  state-changing actions.
- **24 MCP servers** — a large surface area, acknowledged honestly.

## Status

### Works

- Cost routing with a circuit breaker on the local tier (two live tiers
  today, four in the code)
- Cloud tier on an OAuth subscription instead of the API wallet
- Memory 2.0 with a real exponential forgetting curve
- Nightly consolidation at 02:00 (catches up after suspend)
- Circular PyQt6/QWebEngine HUD with live system stats
- Telegram bot (owner-gated, text + voice OGG, persistent reply keyboard,
  confirm tap before mutating actions)
- Obsidian read and write (`vault_read`, `vault_list`, `vault_search`,
  `vault_append`, `vault_daily_append`, `vault_project_append`,
  `vault_decision`)
- Vault semantic search via Telegram — ChromaDB index over Obsidian notes
  (nomic-embed-text embeddings, `vault_recall` tool)
- Spending analysis from bank statements, categorised by merchant type code
- Rotating shift schedule, editable from Telegram, with a pre-shift reminder
- System metrics from one source of truth: averaged and peak temperatures
  with named sensors, fans, GPU, plus a persistent temperature history
- Cost telemetry written to a local JSONL log
- 371 tests across 28 files

### Planned

- Architecture diagram (Mermaid or SVG)
- Demo video or GIF — HUD + Telegram interaction
- A test runner config. There is no `pytest.ini` and no `conftest.py`; the
  suite runs by path, which works and is still a gap

### Removed or disabled

- **Porcupine wake-word.** Disabled, together with the rest of the voice
  stack. Double-clap activation replaced it while voice was still in use.
- **Screen-aware proactivity.** Disabled. ~20 LLM calls per hour with
  rarely actionable hints. Cost not worth the value.
- **Night mode.** Removed on 2026-05-09. Multi-day debug overhead exceeded
  its value.
- **Voice loop (push-to-talk).** Deleted in favor of clap + Telegram voice.
- **Voice, end to end.** Disabled in config (`[voice] enabled = false`,
  `live_mode = false`), not deleted. STT, TTS, and clap activation all
  work; I stopped reaching for them. See the note in
  [What it does](#what-it-does).
- **Cloud vision on photos.** A photo sent to Telegram used to go to a
  cloud vision model. That path spent real money while `local_only = true`
  claimed nothing would — the flag was a lie for exactly one code path.
  Cut. Local vision through Ollama remains.
- **Manual expense tracker.** Five build steps: receipt photo → local model
  extracts the total → category button → JSON. Used once in six months,
  then deleted in favour of reading bank statements.
  [Its own post](blog/en/2026-10-23-deleting-the-feature-i-used-once.md).
- **Career-ops infrastructure** (`_tin_inbox/scans/`, `_tin_inbox/tracked/`).
  Deliberately postponed, not in progress. Mentioned for honesty.

## Failure modes

These are real and they cost real debug time. They are documented because
hiding them would be dishonest.

- **Fake tool calls.** Claude sometimes emits a markdown ` ```shell_exec `
  code block instead of issuing a real `tool_use` block, then claims the
  tool ran. Mitigated with a regex sanitizer in the router. Not a complete
  fix.
- **Local LLM latency >15s.** GPU contention with a parallel content
  pipeline. A circuit breaker opens the local tier for 10 minutes after
  two failures or one slow call, and traffic falls back to Claude.
- **Ollama Xid 31.** Post-suspend GPU MMU fault. Fixed with
  `modprobe nvidia_uvm` and `Restart=always` in systemd.
- **PipeWire audio race.** ALSA EBUSY with WirePlumber. Fixed by
  preferring virtual `pipewire`/`pulse` devices over raw hw.
- **Lid-close double suspend.** logind triggers a second suspend in the
  first 1–2 minutes after resume. Operational, not code.
- **Confirm button bound to nothing.** The ✅ tap on a Telegram message used
  to execute whatever action was pending *right then*, not the action the
  button was attached to — and the buttons kept appearing on every later
  reply, including in a group where other people can press them. Fixed with
  per-action confirm tokens that expire.
- **Currency of the account vs currency of the operation.** In the bank
  statement API the transaction amount and the running balance are in the
  *account's* currency, while the currency code on the same record describes
  the *operation*. Reading them as a pair produces numbers that look
  plausible and are wrong. Cost me a real bug before I read the field
  definitions properly.
- **One request per minute.** The same API rate-limits statement calls hard
  and caps each call to a bounded date range. Naive paging just gets
  refused, so statements are fetched in windows and cached on disk.

The full table is in [ARCHITECTURE.md](ARCHITECTURE.md#failure-modes).

## Reverted decisions

A short list, written in full because architectural reversals are where
the lessons live:

- **Gemma 3 4B → Qwen 3 4B** (2026-04-25). Gemma was unstable under load.
  Qwen has the same footprint and behaves better.
- **Voice loop → clap activation + Telegram voice.** Push-to-talk was
  fragile.
- **Vault writes → read-only on practice, then back to read-write.** The
  privacy window plus the secret scanner initially made writes fail more
  often than they succeeded. Once both were tuned, writing became the normal
  case again — a reversal of a reversal.
- **Night mode → removed entirely** (2026-05-09).
- **Sonnet as the chat model → back to Haiku** (2026-09-17). Routing the
  persona's chat turns through the stronger model made it measurably worse
  at being the persona: the assistant started leaking the identity and
  habits of the coding agent it was being called through. Prompting did not
  suppress it. The right place for a bigger model here is offline analysis,
  not the live conversation.
- **Manual expense tracking → bank statements** (2026-10-17). The feature
  worked. It asked for effort at the one moment nobody has any — right
  after paying for something. Six months, one entry. Deleted.

Details in [ARCHITECTURE.md](ARCHITECTURE.md#decisions-that-were-reverted).

## Hardware

ASUS ROG Strix G713IC. Ubuntu 24.04. NVIDIA RTX 3050 4 GB VRAM.

An upgrade is on the horizon and currently postponed: the 4 GB VRAM
ceiling is the single biggest constraint on local model quality, and the
honest answer to "which card" is still open — more VRAM on an older card
beats fewer gigabytes on a newer one for this workload. See
[ROADMAP.md](ROADMAP.md#hardware-upgrade-horizon).

## Source code access

The code is private. This is not because I am hiding anything — the
architecture is fully documented here and in
[ARCHITECTURE.md](ARCHITECTURE.md).

It is private because:

1. The project contains personal data (memory, vault references, dialog
   logs) intertwined with the code.
2. Cleaning it for public release would take time better spent on the
   project itself.
3. Selective disclosure is the appropriate model for personal
   infrastructure. Open access is one good answer; controlled access is
   another. This project chose the second.

If you are a recruiter, researcher, or engineer with a relevant interest
— AI safety, local-first systems, BCI-adjacent work, agentic systems —
reach out. I am happy to walk you through the code on a call or grant
read access on request.

See [Contact](#contact) below.

## Working with AI as a partner

This project is built in collaboration with Claude Code. Here is what that
means, said plainly.

The architectural decisions are mine. Many of the implementation paths
were suggested by Claude, and I chose among them based on tradeoffs I
could articulate. Where I did not understand the tradeoff well enough,
the result usually shows it — those are the reverted decisions above.

This is a new shape of solo engineering and I am describing it from the
inside. Not "I built this from scratch" — that would be a lie. Not
"Claude built this" — also a lie. The truth: I am an engineer integrating
existing components and AI-suggested patterns into a system that fits my
specific needs, with honest assessment of what I do and do not understand.

## Blog

Technical write-ups in English: [blog/en/](blog/en/).
Українські версії: [blog/uk/](blog/uk/).

Post candidates are listed in [ROADMAP.md](ROADMAP.md#blog-post-backlog).

## Why I'm building this

I'm a veteran of the Armed Forces of Ukraine — 5 years of service across two
periods, including the full-scale invasion.

Local-first AI infrastructure makes sense to me because I have watched
infrastructure fail under conditions where cloud services were either
unreachable or untrustworthy. Building something that runs on a laptop
without depending on OpenAI, Anthropic, or Google staying online and
friendly — that is the point.

### Operational note on public visibility

Beyond the general veteran context above, I keep name, location, unit,
specialization, and operations off the public record. The Russian state has
demonstrated willingness to fund physical harm against Ukrainian veterans
and their families abroad, including documented cases in EU countries
2023–2025. Tightening the public attack surface is operational security,
not modesty.

If you're a researcher, recruiter, or engineer with a relevant interest,
the contact path below filters for context — it isn't there to make you
jump through hoops, it's there because the same filter is what keeps this
project safe to maintain publicly at all.

**Note for legitimate recruiters and collaborators:** I'm happy to discuss
the work, the architecture, and my engineering approach in detail. Personal
details (location, schedule, family, daily patterns) stay private
throughout the conversation — this is policy, not preference. A real
recruitment process never needs that information up front.

## License

MIT. See [LICENSE](LICENSE) for the full text. The license covers the
documentation in this repository and any code snippets that may appear in
blog posts. It does not cover the private source code, which remains
private.

## Contact

**For technical questions about the project:**
Open a [GitHub Issue](../../issues). Public, no introduction needed.

**For research collaboration, recruitment, or anything that needs a
private channel:**
Please introduce yourself first via the contact form:
**[tally.so/r/D4xVL5](https://tally.so/r/D4xVL5)**

A short message describing who you are, what organization you're with,
and what you'd like to discuss is enough. I'll respond with appropriate
contact details from there.

This is not gatekeeping for its own sake — see the operational note in
[Why I'm building this](#why-im-building-this).

**GitHub:** [@frdvch](https://github.com/frdvch)
