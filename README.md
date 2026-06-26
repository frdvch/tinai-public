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

tinAI is a personal voice and text assistant that runs locally on Linux.
It listens (clap activation or Telegram), transcribes Ukrainian speech,
routes the request to the cheapest acceptable model, calls tools through a
local MCP layer, and speaks the answer back. It remembers across days
through a memory system that actually forgets old, unimportant things
instead of accumulating forever.

Core modules:

- **HUD (PyQt6).** Frameless, transparent, always-on-top, draggable.
  Floating in the top-right corner. Shows the transcript, a thinking
  spinner, and live system stats.
- **Voice.** Whisper `small` for STT (CUDA or CPU fallback). Edge TTS in
  Ukrainian Neural voices as the primary speech output, with offline
  Piper as a fallback when there's no network.
- **Memory.** ChromaDB plus a networkx knowledge graph plus an exponential
  forgetting curve. Nightly consolidation at 02:00.
- **Telegram.** Owner-gated bot for remote text and voice (OGG) input,
  with a quick-menu keyboard for Spotify, notes, reminders, system stats,
  and app launching.
- **Obsidian.** Read-only on practice, by design — see
  [ARCHITECTURE.md](ARCHITECTURE.md#obsidian-integration).
- **25 MCP servers** for shell, git, notes, reminders, web search, vision,
  music, vault. Large surface, acknowledged.
- **Cost routing** across Claude Sonnet, Claude Haiku, Groq Llama, and
  local Qwen 3 4B, with a circuit breaker that protects against local LLM
  latency spikes.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the detailed write-up.

Brief overview:

- **4-tier cost routing:** local Qwen 3 4B → Groq Llama → Claude Haiku →
  Claude Sonnet. Order can be inverted with `JARVIS_PREFER_LOCAL=1`.
- **Memory 2.0:** ChromaDB facts/episodes/reflections plus a networkx
  knowledge graph plus an exponential forgetting curve (halflife 30 days,
  threshold 0.1). The decay is real, not planned.
- **PyQt6 transparent HUD** communicating over `ws://localhost:8765`.
- **Read-only Obsidian integration** with secret scanning and a privacy
  window on writes.
- **Telegram remote access** for text and voice.
- **25 MCP servers** — a large surface area, acknowledged honestly.

## Status

### Works

- 4-tier cost routing with a circuit breaker on the local tier
- Memory 2.0 with a real exponential forgetting curve
- Nightly consolidation at 02:00 (catches up after suspend)
- PyQt6 HUD with transcript, thinking spinner, and live system stats
- Telegram bot (owner-gated, text + voice OGG + quick-menu keyboard)
- Obsidian read access (`vault_read`, `vault_list`, `vault_search`,
  `vault_daily_append`)
- Clap activation as a wake mechanism
- Vault semantic search via Telegram — ChromaDB index over Obsidian notes
  (nomic-embed-text embeddings, `vault_recall` tool)
- Cost telemetry written to `~/.local/share/jarvis/tool_stats.jsonl`
- Edge TTS primary, Piper fallback

### Planned

- HUD refactor — remove transcript panel, keep stats bar and profile
  toggle only
- Architecture diagram (Mermaid or SVG)
- Demo video or GIF — HUD + Telegram interaction

### Removed or disabled

- **Porcupine wake-word.** Disabled. The API key was never set. Clap
  activation replaces it.
- **Screen-aware proactivity.** Disabled. ~20 LLM calls per hour with
  rarely actionable hints. Cost not worth the value.
- **Night mode.** Removed on 2026-05-09. Multi-day debug overhead exceeded
  its value.
- **Voice loop (push-to-talk).** Deleted in favor of clap + Telegram voice.
- **Career-ops infrastructure** (`_tin_inbox/scans/`, `_tin_inbox/tracked/`).
  On paper only. The Python side is an empty stub. The vault folders do
  not exist yet. Mentioned for honesty.

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

The full table is in [ARCHITECTURE.md](ARCHITECTURE.md#failure-modes).

## Reverted decisions

A short list, written in full because architectural reversals are where
the lessons live:

- **Gemma 3 4B → Qwen 3 4B** (2026-04-25). Gemma was unstable under load.
  Qwen has the same footprint and behaves better.
- **Voice loop → clap activation + Telegram voice.** Push-to-talk was
  fragile.
- **Vault writes → read-only on practice.** The privacy window plus the
  secret scanner made writes fail more often than they succeeded.
- **Night mode → removed entirely** (2026-05-09).

Details in [ARCHITECTURE.md](ARCHITECTURE.md#decisions-that-were-reverted).

## Hardware

ASUS ROG Strix G713IC. Ubuntu 24.04. NVIDIA RTX 3050 4 GB VRAM.

Upgrade on the horizon: RTX 5060 Ti 16 GB, if the project hits the VRAM
ceiling with larger local models (Qwen 7B+ or local embeddings). See
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

Post candidates are listed in [ROADMAP.md](ROADMAP.md#blog-post-candidates).

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
**LinkedIn:** [linkedin.com/in/antshkro](https://linkedin.com/in/antshkro)
