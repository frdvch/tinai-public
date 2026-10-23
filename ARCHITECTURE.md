# Architecture

This document describes how tinAI is put together. It is written from the
inside, with concrete details, and it names the things that do not work or
were removed. The source code is private (see [README.md](README.md) on
selective disclosure), but the architecture is fully described here.

## Footprint

| Metric | Value |
|---|---|
| Source files | 136 |
| Lines of code | ~28,800 (Python + JS + HTML + CSS + shell) |
| Repo size (no .git, no venv) | 16 MB |
| Largest modules | `core/agent.py` 2367 LOC, `core/telegram_listener.py` 2301 LOC, `hud/window.py` 1208 LOC, `main.py` 1104 LOC, `core/router.py` 1053 LOC |
| Tests | 371 test functions across 28 files |

Counted 2026-10-23. `hud/window.py` is the legacy HUD, kept but no longer
the one that launches — see [HUD](#hud).

## Hardware

- ASUS ROG Strix G713IC laptop
- Ubuntu 24.04
- NVIDIA RTX 3050, 4 GB VRAM
- Consumer hardware is a deliberate constraint, not a limitation to apologize
  for. The point is that the assistant should be runnable for someone who
  cannot afford or does not want a server.

## Data flow

The live path on a typical interaction. Telegram text is the entry point
that actually carries traffic; the audio branch is intact but switched off
in config.

```
   Telegram (text)          microphone  [DISABLED: voice.enabled = false]
          |                      |
          |            +-------------------+
          |            | voice/listener.py |  clap detector / wake-word
          |            +---------+---------+
          |                      |
          |            +-------------------+
          |            |   voice/stt.py    |  faster-whisper, uk
          |            +---------+---------+
          |                      |
          +----------+-----------+
                     v
           +----------------------+
           | core/classifier.py   |  pattern early exits, instant replies
           +---------+------------+
                     v
           +----------------------+
           |   core/router.py     |  provider selection (see Tier routing)
           +---------+------------+
                     v
           +----------------------+
           |    core/agent.py     |  ReAct loop, tool dispatch,
           |  (core/react.py)     |  memory injection, confirm gate
           +---------+------------+
                     |
          +----------+-----------------+
          v                            v
  core/claude_code_client.py        Ollama
  `claude -p` on OAuth sub        (Qwen 3 4B, local)
  (Haiku; no tools, no MCP)
          |                            |
          +----------+-----------------+
                     v
           +----------------------+     +----------------------+
           |  core/ws_server.py   | --> |   core/web_hud.py    |
           |  ws://localhost:8765 |     | circular QWebEngine  |
           |  origin allowlist    |     | over :8766 static    |
           +----------------------+     +----------------------+
                     |
                     v
              Telegram reply
```

The dormant branches — Groq, Claude Sonnet over the API, and `voice/tts.py`
(Edge TTS primary, Piper offline fallback) — are still wired in the code and
unreachable in the current configuration.

## Tier routing

The router is the part that decides where a request goes. It lives in
`core/router.py` (~1050 LOC).

Tiers as designed, in cost order:

1. **Qwen 3 4B instruct** — local Ollama, fits in 4 GB VRAM. $0 per call.
2. **Groq Llama 3.3 70B** — Groq API. ~$0.59 input / $0.79 output per Mtok.
3. **Claude Haiku 4-5** — ~$1 input / $5 output per Mtok at API prices.
4. **Claude Sonnet 4-6** — ~$3 input / $15 output per Mtok at API prices.

### What actually runs

`local_only = true` is the current state, and under it `_choose_provider`
collapses the ladder to two live tiers:

| Request shape | Chain |
|---|---|
| Tools available, short imperative | local read-only tier, no fallback |
| Tools available, anything else | Haiku → local read-only |
| No tools (chat, journal) | Haiku → local |

Groq and Sonnet are never selected in this state. The four-tier chains are
still in the code and come back when `local_only = false`: simple query →
`local → Claude → Groq`; short tool intent → `Haiku → Claude → Groq`;
default → `Claude → Groq → local`.

Two environment overrides remain: `JARVIS_PREFER_LOCAL` (defaults to on) and
`JARVIS_HAIKU_DISABLE`.

One safety rule in `router.chat`: if tools were requested and the cloud tier
failed on authentication, the remaining providers are **skipped** rather than
allowed to answer. A provider without tools, asked to do something that needs
tools, invents the result — and a fabricated tool result poisons memory.
Better to fail loudly.

### Subscription transport

The Haiku tier does not go through the API wallet. `core/claude_code_client.py`
shells out to the `claude -p` CLI on an OAuth subscription
(`haiku_transport = "subscription"` in config) and parses its JSON output.

The subprocess is deliberately boxed in:

- zero MCP servers and zero tools (`--strict-mcp-config` with an empty server
  map, `--allowedTools ""`), so it cannot touch the machine;
- a neutral working directory, so it does not pick up project context;
- `ANTHROPIC_API_KEY` stripped from its environment — otherwise the CLI
  silently bills the API key instead of the subscription;
- its own process group, killed wholesale on a 90-second timeout;
- any failure raises rather than returning a half-answer.

This is the main reason the cloud tier survives a month with no API budget.
The tradeoff is honest: it is a CLI subprocess, not a supported API surface,
and it can break whenever that CLI changes.

### Local circuit breaker

The local tier has a circuit breaker in `core/router.py`. It opens when:

- Two consecutive Qwen calls fail, or
- A single call returns slower than 15 seconds.

The breaker stays open for 10 minutes. While open, all requests skip
straight to Claude. The breaker exists because the same machine runs a
parallel internal pipeline, and the GPU gets contended.

### Hardcoded pricing

Cost is estimated locally using hardcoded per-Mtok rates. There is no
billing API integration. The numbers are the public list prices as of
spring 2026. Cache discount (10% read, 1.25x write) is also estimated, not
validated against actual invoices.

### Fake tool call sanitizer

Sometimes Claude responds with a markdown code block describing a tool
call instead of issuing a real `tool_use` block. Then it claims the tool
ran. The sanitizer in `core/router.py` detects and strips those markdown
blocks before falling back. It is not a complete fix — the underlying
model behavior is what it is — but it catches the common cases.

## Memory 2.0

Memory has three layers, all in `core/memory*.py`.

### Layer 1: ChromaDB (semantic)

`core/memory.py`, ~360 LOC. Three collections:

- **facts** — auto-extracted preferences, habits, work patterns
- **episodes** — daily summarized interactions
- **reflections** — output of nightly consolidation

Stored at `data/chromadb/`. Search uses ChromaDB cosine similarity. No
custom relevance scoring on top.

### Layer 2: Knowledge graph (structured)

`core/memory_graph.py`, ~221 LOC. A networkx graph pickled to
`data/graph.pickle`. Nodes are entities (people, projects, tools), edges
are relationships. The graph is built incrementally by the nightly
consolidation job, not from raw chat in real time.

### Layer 3: Forgetting curve

Real, not planned. Implemented as exponential decay:

```
weight = exp(-Δdays / halflife)
```

- halflife = 30 days
- threshold = 0.1 (anything below is dropped on next consolidation)

The decay runs in the nightly consolidation pass. Decay applies to
ChromaDB facts. Graph edges currently do not decay — this is a known gap.

### Nightly consolidation

`core/night_agent.py` runs at 02:00 local time. If the machine was
suspended at 02:00 it catches up on next wake. Steps:

0. Facts are no longer *discovered* here: extraction runs in the background
   during the conversation, because waiting until 02:00 meant forgetting
   something said an hour earlier. The nightly job aggregates what the day
   collected.
1. Aggregate the day's raw facts into episodes.
2. Extract entities and relations from new episodes into the graph.
3. Apply decay to ChromaDB facts.
4. Write a summary to `data/last_consolidation.json`.
5. On Sundays, write a weekly digest into the Obsidian vault under
   `vault/learned/`.

## HUD

Two HUD implementations exist in the tree. The circular web HUD is the one
that launches; `hud/window.py` is the earlier panel UI, kept and unused.

### Circular HUD

`core/web_hud.py` — a PyQt6 shell around a `QWebEngineView`, rendering a web
UI served locally by `core/static_server.py` on port 8766.

Window properties:

- `FramelessWindowHint`
- `WA_TranslucentBackground`
- `WindowStaysOnTopHint`
- Draggable by the body, because there is no native title bar to grab
- Ctrl+wheel to resize
- Width from config (380 px by default), positioned with a gap from the edge

What it shows: a concentric-ring central circle with a glow, state colour,
and live system stats. What it deliberately does **not** show: a chat input
and a transcript panel. Both existed in the panel HUD and both were removed
in the 2026 redesign — all input routes through Telegram or voice, and the
HUD became a status surface rather than a conversation window.

That removal closed the "HUD refactor" item that older versions of this
document listed as planned. The reasoning-visibility gap below stayed open.

### Reasoning visibility — honest gap

The HUD shows state and a generic thinking indicator during tool loops. It
does **not** visualize intermediate steps: which tool was called, what
arguments, what came back. The full log is written to a local JSONL dialog
log, but reading it is post-hoc.

Adding step-by-step reasoning visibility is on the wishlist. It is named
here because hiding it would be dishonest about the state of the project.

### WebSocket protocol

`core/ws_server.py`. Endpoint `ws://localhost:8765`. Messages are JSON:

| `type` | Meaning |
|---|---|
| `message` | Chat message from user or AI |
| `stats` | CPU/RAM/temp/GPU/battery snapshot |
| `profile` | Hardware profile change (silent / loud / etc) |
| `mode` | AI mode toggle (Gemini ↔ Claude) |

The socket also accepts an `input` message — a line of text pushed into the
agent from the HUD. That direction is the reason the server now enforces an
origin allowlist; see [Security hardening](#security-hardening).

Thread safety: handlers use `asyncio.run_coroutine_threadsafe` to bridge
non-async producers. On reconnect, the server replays sticky state (last
mode, last profile).

## Voice I/O

**Current state: off.** `[voice] enabled = false` and `live_mode = false` in
config. Everything below is present and working code that nothing currently
calls, except Whisper, which still transcribes voice messages arriving
through Telegram.

Why it is off: voice was the founding premise of the project and became the
least-used part of it. Typing into Telegram from a phone beats speaking at a
laptop in nearly every situation that actually comes up — and the quiet
version of that admission would have been to leave the README claiming a
voice assistant.

### STT

`voice/stt.py`. faster-whisper, language `uk`, auto-detects CUDA, float16 on
GPU with an int8 CPU fallback. Model size was raised from `tiny` to `small`
after `tiny` could not handle mixed Ukrainian/Russian speech. A separate
test of `medium` confirmed it is usable for Ukrainian dictation: the meaning
survives, names and job titles do not, and there is no diarization.

### TTS

`voice/tts.py`, ~313 LOC.

- Primary: Edge TTS (Microsoft) with Ukrainian Neural voices (`Ostap`,
  `Polina`). Online, free, very natural. Requires network.
- Fallback: Piper with the `uk_UA-lada-x_low` model. Fully offline. Robotic
  but functional.

The TTS module also extracts an RMS amplitude envelope to drive the HUD's
voice visualization during playback.

### Wake-word (disabled)

`voice/listener.py` integrates Porcupine for wake-word detection. It is off
with the rest of the voice stack. The config also carries a list of fuzzy
spellings of the wake word, because Whisper transcribes a short Ukrainian
word a dozen different ways and an exact match loses most activations.

### Clap detector

`core/clap_detector.py` listens for a sharp clap on the microphone and
triggers activation. Simple amplitude threshold with a debounce. Good
enough for a personal device, not the right answer for production.

## Obsidian integration

`mcp_servers/obsidian_server.py`. Eight tools:

- `vault_read(path)` — reads a markdown file
- `vault_list(path)` — lists a folder
- `vault_search(query)` — substring search
- `vault_write(path, text)` — replaces a file
- `vault_append(path, text)` — appends to a file
- `vault_daily_append(text)` — writes to today's daily note
- `vault_project_append(project, text)` — appends to a project note
- `vault_decision(...)` — records a decision with its context

Earlier versions of this document said the vault was "read-only on
practice". That is no longer true: the write paths are used daily. What
guards them:

- A **privacy window** (`core/privacy.py`): while it is open, vault writes
  and verbatim logging are suspended.
- A **secret scanner**: writes containing patterns like `sk-ant-`, `sk-`,
  `AIza`, `ghp_`, `gsk_`, `tvly-` are rejected. It deliberately does not
  block ID-shaped or money-shaped numbers yet.

### Two path resolvers

Reads and writes resolve paths differently, and the asymmetry is deliberate:

- `_resolve()` — the write path. Resolves symlinks, then asserts the result
  is inside the vault root.
- `_resolve_read()` — rejects absolute paths, rejects `..` in any component,
  and intentionally skips symlink resolution, so that deliberate symlinks
  pointing out of the vault keep working.

The asymmetry exists because of a bug worth naming: in Python,
`VAULT_ROOT / "/etc/hostname"` collapses to `/etc/hostname`. Read tools are
not in the mutating set, so they need no confirmation — which meant
`vault_read` would quietly read any file on the machine. Fixed, and pinned
by a test.

The vault paths the agent reads from:

- `vault/daily/` — daily notes for context
- `vault/projects/` — ongoing work
- `vault/decisions/` — decision history (written by the router on auth
  failures, only)
- `vault/learned/` — weekly digests
- `vault/claude-memory` — symlink to the persistent Claude memory directory

A separate watchdog daemon (`core/vault_sync.py`) syncs the vault to a
private git remote when enabled. Auto-push is `false` in config by default.

### Vault semantic search

`core/vault_index.py` (~187 LOC) and `core/vault_search.py` (~32 LOC).

The vault is indexed into a ChromaDB collection using `nomic-embed-text`
embeddings via the local Ollama instance. The index is stored alongside the
main memory ChromaDB database, on the order of hundreds of chunks.

The agent exposes a `vault_recall` tool. When called with a natural language
query, it runs cosine similarity search against the index and returns the top
matching chunks with their source paths. The result is synthesized by the
local Qwen model before returning to the user.

The index is not automatically rebuilt on vault changes — rebuilds are
triggered manually or via the nightly consolidation job.

## Telegram bot

`core/telegram_listener.py`, ~2300 LOC. Owner-gated: an `OWNER_ID` constant
filters every callback. All non-owner messages are silently ignored.

Capabilities:

- Text in private chat → `agent.respond()`
- Voice OGG → Whisper → `agent.respond()`; the audio file is archived, not
  deleted
- Group topic routing: topic IDs configured in `config.toml`
- A persistent, hierarchical reply keyboard
- Inline buttons for per-item actions (one note, one reminder, one app)
- A confirm tap before any state-changing action
- `setChatMenuButton` — the "/" button in the text field shows only `/menu`

### Menu structure (as of 2026-10)

The keyboard is a tree of levels, not a flat row of shortcuts. The top level:

| Button | Opens |
|---|---|
| Програми | Desktop apps, launched via `app_launch()` |
| Запущені | Running apps, built dynamically; status or close per app |
| Скріншот | Screen picker, then capture and describe |
| Робота | Work schedule: week and month view, shift swap |
| Продуктивність | Notes and reminders, each with inline per-item actions |
| Фінанси | Balances and spending summaries from bank statements |
| Spotify | Inline transport: ⏮ ⏯ ⏭, volume ±10, current track |
| Утиліти | Weather, clipboard read/write, timer |
| Показники | Temperatures, fans, load, GPU, battery — see below |
| Допомога | What the buttons do |

Two implementation notes that cost debugging time:

- Reply-keyboard buttons arrive as plain text, so the menu level has to be
  remembered per chat and **persisted to disk** — the service restarts and an
  in-memory level would strand the user one tap deep in a tree.
- A reverse index maps every button label to its level, so a button still
  works when the stored state is lost. Without it, taps leaked through to the
  LLM, which then cheerfully answered as if "📊 Показники" were a question.

### System metrics

`core/sysmetrics.py` is the single source of truth behind the Показники
button and the `system_status` MCP tool. It used to be two divergent copies,
one in the Telegram handler and one in the MCP server, and the number they
reported was `max()` over every sensor the machine exposes — including the
WiFi card. A single instantaneous sample of a laptop that idles anywhere
between 62 °C and 92 °C is also close to meaningless.

It now reports averages and peaks over a short window, names each sensor,
adds the NVIDIA GPU (absent from `sensors` output) and fan speeds.
`core/thermal_history.py` reads a persistent minute-by-minute temperature
log, so "hotter than last week?" has an answer, and the night's summary
lands in the morning briefing.

### Spotify control

Spotify is controlled via D-Bus directly (`dbus-send` to
`org.mpris.MediaPlayer2.spotify`), without the Spotify Web API or OAuth.
This covers play/pause/next/previous and volume adjustment. If the
`spotify` process is not running, the bot launches it first via
`app_launch()` and then opens the transport keyboard.

The bot is opt-in (`tg.enabled = false` by default in config).

## Finance from bank statements

`core/mono.py` is a read-only client for a bank's personal API: account
balances and statements. `core/mono_analytics.py` turns statements into
monthly totals, category breakdowns, comparisons between periods, detected
subscriptions, and loan movements. Exposed through two MCP servers
(`mono_server.py`, `finance_server.py`), with 85 tests behind them.

Three design decisions worth stating:

**Categories come from merchant type codes, not from a model.** The earlier
version asked a local model to categorise each transaction. It hallucinated
both categories and currencies. A deterministic lookup from the merchant
category code is less clever, always the same answer twice, and cheaper than
any model call. Not every problem near an assistant is an LLM problem.

**The API rate-limits hard** — roughly one statement request per minute, with
each request capped to a bounded date range. Naive paging just gets refused.
Statements are fetched in windows and cached on disk, outside the repo.

**Amount and currency describe different things.** In a statement record the
amount and the running balance are denominated in the *account's* currency,
while the currency code on the same record describes the *operation*. Read
as a pair, they produce numbers that look plausible and are wrong. This cost
a real bug before I read the field definitions properly instead of guessing
from the names.

What this replaced: a manual receipt tracker, built over five steps —
photograph a receipt, let a local vision model pull the total, tap a
category, store it. It worked, and it was used once in six months. The bank
already knows every card transaction, including a merchant type code, so the
feature was deleted rather than polished.

## Work schedule

A rotating shift schedule lives as a note in the vault. `core/schedule.py`
renders week and month views, `core/schedule_edit.py` writes swaps back from
Telegram, and a reminder fires before each shift with an acknowledge button
and follow-up nudges if it is not acknowledged.

Two things here were only found by using it: night shifts crossing midnight
have to be attributed to the date they *end* on, or a month boundary silently
loses a shift; and timezone conversion after moving country has to be
computed relative to the machine's zone **on the date of the change**, not
the current one, or every historical shift shifts by an hour.

## MCP servers

There are 24 MCP servers under `mcp_servers/`. This is a large surface area
and it is acknowledged. Some are heavily used, some rarely. All of them
register the same way: an `mcp.server.Server` over stdio with an explicit
tool list.

| Server | Purpose |
|---|---|
| `obsidian_server.py` | Vault read and write (8 tools) |
| `mono_server.py` | Bank accounts and statements, read-only |
| `finance_server.py` | Spending analysis over those statements |
| `shell_server.py` | Shell command execution, scoped by allowlist |
| `git_server.py` | Local git operations |
| `notes_server.py` | Personal notes I/O |
| `reminders_server.py` | Time-based reminders |
| `web_server.py` | Search, weather, translation |
| `news_server.py` | Headlines |
| `vision_server.py` | Screen capture + describe (local or cloud model) |
| `spotify_server.py` | Spotify control |
| `apps_server.py` | Launch desktop apps, media keys |
| `windows_server.py` | Window list, focus, layout, tab switching |
| `system_server.py` | System status, thin wrapper over `core/sysmetrics.py` |
| `rog_server.py` | Laptop power profile and GPU state |
| `network_server.py` | Link status, throughput test, per-process traffic |
| `health_server.py` | Self-diagnostic across the stack |
| `balance_server.py` | Remaining credit on each LLM provider |
| `clipboard_server.py` | Clipboard read and history search |
| `docs_server.py` | Project documentation lookup |
| `projects_server.py` | Project inventory and structure |
| `code_agent_server.py` | Hands a coding task to a coding agent |
| `ml_router_server.py` | Dormant — generic ML task dispatch from an earlier project |
| `yolo_server.py` | Dormant — object detection metrics from an earlier project |

The last two are honest dead weight: they belong to a computer-vision
project that is no longer active. If the project gets pruned, the MCP layer
is the first place I would start, and those are the first two files.

## External dependencies

From `requirements.txt`:

- `anthropic>=0.40.0`
- `groq>=0.13.0`
- `ollama>=0.4.0`
- `chromadb>=0.5.0`
- `networkx>=3.2`
- `mcp>=1.0.0`
- `PyQt6>=6.7.0`
- `mss>=9.0`, `pillow>=10.0` (screen capture)
- `duckduckgo-search>=6.0.0`
- `psutil>=6.0.0`
- `rich>=13.0.0`
- `requests>=2.32.0`
- `watchdog>=3.0`
- `pytest>=8.0`

Missing from `requirements.txt` but imported in code (a real bug, still
open, re-verified 2026-10-23):

- `websockets` — the HUD socket
- `aiohttp` — the local static server
- `pvporcupine`, `sounddevice` — voice input
- `faster-whisper`, `piper-tts` — STT and offline TTS
- `playwright` — browser automation

Not missing, contrary to what this file used to claim:
`python-telegram-bot` is not imported anywhere. The Telegram integration
long-polls `getUpdates` with `requests`, which is already listed. The old
entry was a guess from the module name, which is exactly the kind of thing
this document is supposed to catch.

## Security hardening

Two rounds of deliberate bug-hunting across the codebase (2026-10) produced
most of this list. It is written down because a personal assistant with
shell access, a bank connector and a chat interface is a small attack
surface with a large blast radius, and because the fixes are more
interesting than the absence of bugs would have been.

**Path validation on vault reads.** `VAULT_ROOT / "/etc/hostname"` collapses
to `/etc/hostname` in Python. Read tools are not in the mutating set, so
they need no confirmation — which meant the read tool would quietly return
any file on the machine. Fixed with a separate read resolver that rejects
absolute paths and `..` components. See
[Two path resolvers](#two-path-resolvers).

**Origin allowlist on the HUD websocket.** The socket accepts an `input`
message that goes straight into the agent. It listened on localhost with no
origin check, so any web page open in a browser on the same machine could
connect and inject instructions. Now the server passes an explicit origin
allowlist.

**Per-action confirm tokens.** State-changing tools require a confirm tap in
Telegram. The tap used to execute whatever action was pending *at that
moment*, not the one the button belonged to, and the buttons kept appearing
under later replies — including in a family group where other people can
press them. Each pending action now carries a random token with a
ten-minute lifetime, and the tap is only honoured for its own action.

**Secrets outside the repo and off NTFS.** The `.env` file lives under
`~/.config/jarvis/` on an ext4 filesystem with restrictive permissions,
never in the project tree. It moved there after a specific discovery:
`chmod` on an NTFS volume mounted via `ntfs-3g` returns success and does
nothing — the whole volume is world-readable regardless. Five call sites and
four hardcoded paths had to be repointed. Precedence is documented as
environment over config file, and `.env.example` carries placeholders only.

**Redaction before persistence.** `core/redactor.py` scrubs
credential-shaped strings before anything is written to memory or logs,
because the thing most likely to leak a key is not an attacker — it is the
assistant helpfully quoting a config file back into its own long-term
memory.

**Subprocess scoping.** Shell execution is limited to three allowed
directories with a 30-second timeout. The subscription transport runs with
no tools, no MCP servers, a neutral working directory, and its own process
group.

No exploit details here, and no claim that this list is complete. It is the
set of holes I found and closed, pinned by a small dedicated test file so
they stay closed.

## Failure modes

These are real and recurring. Each one cost real debug time.

| Failure | Cause | Mitigation |
|---|---|---|
| Fake tool calls | Claude emits a markdown ` ```shell_exec ` block instead of `tool_use` and then claims it ran | regex sanitizer in router; not complete |
| Local LLM latency >15s | GPU contention with a parallel internal pipeline | circuit breaker opens for 10 min |
| Ollama Xid 31 | post-suspend GPU MMU fault on NVIDIA driver | `modprobe nvidia_uvm` + systemd `Restart=always` |
| PipeWire audio race | ALSA EBUSY with WirePlumber on raw hw devices | prefer virtual `pipewire`/`pulse` devices |
| Lid-close double suspend | logind triggers a second suspend in the first 1–2 min after resume | avoid lid-close within that window |
| Stale Gemini session | `1008 BidiGenerateContent` on the live audio socket | delete the session file, restart |
| Confirm tap bound to nothing | ✅ executed the action pending at that moment, not the one the button belonged to | per-action tokens with a 10-minute lifetime |
| Statement currency misread | amount and balance are in the account's currency; the currency code on the same record describes the operation | read the field definitions, not the field names |
| Statement API refuses paging | ~1 request per minute, each capped to a bounded date range | fetch in windows, cache on disk |
| Cloud vision billed under `local_only` | one code path sent Telegram photos to a cloud model while the flag promised no cloud calls | path removed; local vision retained |
| Reply-keyboard taps leaking to the LLM | plain-text buttons with menu state held only in memory, lost on restart | persist menu state to disk + reverse index from label to level |

## Decisions that were reverted

This list is shorter than I'd like, but every item taught something.

- **Gemma 3 4B → Qwen 3 4B** (2026-04-25). Gemma was unstable under load
  on the 4GB VRAM ceiling, with 30–150 s response times and one 8-hour
  downtime from a GPU cascade. Qwen 3 4B has the same footprint, better
  Ukrainian, and stronger tool calling. The config section is still named
  `[llm.gemma]` and `_call_gemma` still exists in the router — a separate
  refactor.
- **Voice loop → clap activation + Telegram voice** (April 2026). Push-to-
  talk loop was fragile and rarely the right interaction. Clap activation
  is cruder but more reliable. Voice in Telegram covers the remote case.
- **Night mode removed** (2026-05-09). Multi-day debug overhead exceeded
  the value the feature provided. Only the leaf function
  `session.night_mode()` remains.
- **Vault writes → read-only on practice** (early 2026), **and back again**
  (autumn 2026). The privacy window plus secret scanner initially made write
  paths fail more often than they succeeded, so the vault became a read
  source. Once the guards were tuned and the sync daemon was fixed, writing
  became the normal case again. Listed as a reversal of a reversal, because
  that is what it was.
- **Sonnet as the chat model → back to Haiku** (2026-09-17). Routing the
  persona's conversational turns through the stronger model made it
  measurably worse *at the role*: it began leaking the identity and habits
  of the coding agent it was invoked through — plan-mode phrasing, tooling
  talk, the wrong name for itself. Prompting did not suppress it. The right
  job for a larger model here is offline analysis of the conversation, not
  the conversation.
- **Manual expense tracking → bank statements** (2026-10-17). Built in five
  steps, used once in six months, deleted. The failure was not technical:
  the feature asked for effort at the exact moment nobody has any. See
  [Finance from bank statements](#finance-from-bank-statements).
- **LLM-assigned spending categories → merchant type codes** (2026-10-17).
  The model hallucinated categories and currencies. A deterministic lookup
  is duller and correct.
- **Nightly memory consolidation as the only extraction pass → per-turn
  background extraction.** Waiting until 02:00 to notice a fact meant the
  assistant forgot things said an hour earlier. Facts are now extracted in
  the background during the conversation; the nightly job still aggregates,
  builds the graph and applies decay.

## Testing

**371 test functions across 28 files** in `tests/` (counted 2026-10-23).
Earlier versions of this document said "three test files", which was true
when it was written and badly out of date by the time anyone read it.

The largest groups, which map neatly onto where the bugs actually were:

| Area | Files | Rough size |
|---|---|---|
| Bank statements and spending analysis | `test_mono.py`, `test_mono_analytics.py`, `test_finance_*.py` | ~85 tests |
| Provider routing | `test_router_smart.py` | ~38 tests |
| Shift schedule and pay | `test_schedule*.py`, `test_payroll.py` | ~57 tests |
| Audit regressions | `test_audit_fixes.py`, `test_audit2_*.py` | ~60 tests |
| Shopping list and Telegram menus | `test_shopping*.py`, `test_menu_buttons.py`, `test_finance_menu.py` | ~40 tests |
| Memory, metrics, thermal history, voice archive | `test_memory_v2.py`, `test_sysmetrics.py`, `test_thermal_history.py`, `test_voice_archive.py` | ~45 tests |

`tests/test_audit2_security.py` is small and deliberate: it pins the two
security fixes described in [Security hardening](#security-hardening), so a
future refactor cannot quietly reopen them.

Most of these tests are regression guards written *after* a bug, not
specifications written before one. That is the honest description of how
this suite grew.

Two gaps, named rather than hidden:

- **No test runner config.** No `pytest.ini`, no `pyproject.toml`, no
  `conftest.py`. The suite runs by path inside the virtualenv. It works and
  it is not a setup I would hand to anyone else.
- **No end-to-end tests.** Nothing drives the real Telegram API, a real
  model, or the audio path. Those will land when an integration regression
  costs enough debug time to pay for them.

## What the code is not

For completeness:

- Not a framework. There is no plugin system, no third-party extension
  contract.
- Not a research project. There are no novel ML contributions.
- Not multi-user. There is one `OWNER_ID` and one local profile.
- Not portable today. Hardcoded paths to a user-specific external disk
  mount exist in several files and would need to be parameterized before
  someone else could run this unchanged.
