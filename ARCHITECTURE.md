# Architecture

This document describes how tinAI is put together. It is written from the
inside, with concrete details, and it names the things that do not work or
were removed. The source code is private (see [README.md](README.md) on
selective disclosure), but the architecture is fully described here.

## Footprint

| Metric | Value |
|---|---|
| Source files | 101 |
| Lines of code | ~22,600 (Python + JS + HTML + CSS + shell) |
| Repo size (no .git, no venv) | 7.3 MB |
| Largest modules | `core/agent.py` 1886 LOC, `main.py` 1274 LOC, `hud/window.py` 1217 LOC, `core/telegram_listener.py` ~1150 LOC, `core/router.py` ~830 LOC |

## Hardware

- ASUS ROG Strix G713IC laptop
- Ubuntu 24.04
- NVIDIA RTX 3050, 4 GB VRAM
- Consumer hardware is a deliberate constraint, not a limitation to apologize
  for. The point is that the assistant should be runnable for someone who
  cannot afford or does not want a server.

## Data flow

The voice-to-voice path on a typical interaction:

```
                 +-------------------+
  microphone --> | voice/listener.py |  Porcupine wake-word (disabled)
                 |                   |  or core/clap_detector.py (active)
                 +---------+---------+
                           |
                           v
                 +-------------------+
                 |   voice/stt.py    |  faster-whisper "small"
                 |                   |  CUDA float16 / CPU int8
                 +---------+---------+
                           |
                           v
                 +----------------------+
                 | core/classifier.py   |  pattern-based early exits
                 +---------+------------+
                           |
                           v
                 +----------------------+
                 |   core/router.py     |  4-tier provider selection
                 +---------+------------+
                           |
                           v
                 +----------------------+
                 |    core/agent.py     |  ReAct loop, tools
                 |  (core/react.py)     |  memory injection
                 +---------+------------+
                           |
                +----------+-----------+
                |          |           |
                v          v           v
            Anthropic   Groq       Ollama
            (Sonnet     (Llama     (Qwen 3 4B,
             / Haiku)    3.3 70B)   local)
                |          |           |
                +----------+-----------+
                           |
                           v
                 +----------------------+
                 |    voice/tts.py      |  Edge TTS (primary)
                 |                      |  Piper (fallback, offline)
                 +---------+------------+
                           |
                           v
                 +----------------------+
                 |  core/ws_server.py   |  ws://localhost:8765
                 +---------+------------+
                           |
                           v
                 +----------------------+
                 |   hud/window.py      |  PyQt6 transparent HUD
                 +----------------------+
```

There is also a Telegram path (text or voice OGG) that bypasses the audio
listener and feeds straight into the agent.

## Tier routing

The router is the part that decides where a request goes. It lives in
`core/router.py` (~818 LOC).

Tiers, in cost order:

1. **Qwen 3 4B q8_0** — local Ollama (8-bit quantized, 4.4 GB, fits fully in 4 GB VRAM). $0 per call.
2. **Groq Llama 3.3 70B** — Groq API. ~$0.59 input / $0.79 output per Mtok.
3. **Claude Haiku 4-5** — Anthropic API. ~$1 input / $5 output per Mtok.
4. **Claude Sonnet 4-6** — Anthropic API. ~$3 input / $15 output per Mtok.

Selection logic in `_choose_provider`:

- If the query is short (<80 chars), info-shaped, and has no action verb,
  classify as `simple_query` and prefer the local tier.
- If the query is short and looks like a tool intent (action verb, no
  complexity markers), prefer the Haiku tier with tools enabled.
- Otherwise route to Sonnet with tools enabled.

The order can be inverted by setting `JARVIS_PREFER_LOCAL=1`, which makes
the router try Qwen first regardless of classification.

A `local_only = true` flag in `[llm.local]` disables cloud fallback entirely.
When set, `_choose_provider` returns only the local tier — useful when API
credits are unavailable. Toggle back with `local_only = false` and restart.

### Local circuit breaker

The local tier has a circuit breaker (`router.py:18-48`). It opens when:

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
ran. The sanitizer (`router.py:51-137`) detects and strips those markdown
blocks before falling back. It is not a complete fix — the underlying
model behavior is what it is — but it catches the common cases.

## Memory 2.0

Memory has three layers, all in `core/memory*.py`.

### Layer 1: ChromaDB (semantic)

`core/memory.py`, ~348 LOC. Three collections:

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

1. Aggregate the day's raw facts into episodes.
2. Extract entities and relations from new episodes into the graph.
3. Apply decay to ChromaDB facts.
4. Write a summary to `data/last_consolidation.json`.
5. On Sundays, write a weekly digest into the Obsidian vault under
   `vault/learned/`.

## HUD

Two HUD implementations exist. The PyQt6 one is the default.

### PyQt6 HUD

`hud/window.py`, ~1217 LOC.

Window properties:

- `FramelessWindowHint`
- `WA_TranslucentBackground`
- `WindowStaysOnTopHint`
- Draggable via custom mouse event handlers
- Position: top-right, `x = screen_width - 420 - 20`, `y = 30`
- Size: 420×700

Display elements:

- Title "SHADOW" in yellow bold
- 8 px cyan blinking state indicator
- Transcript bubbles (cyan for user, yellow for AI)
- Thinking spinner with "THINKING ···" animation (80 ms tick)
- Stats bar: CPU, RAM, temperature, GPU, battery
- Text input + submit
- System tray icon (yellow diamond 32×32)
- Scanlines overlay (4 px spacing, 3% opacity)
- Corner decorators (16 px yellow lines, 50% opacity)
- Single-switch AI toggle in the top bar (Gemini ↔ Claude)

### Reasoning visibility — honest gap

The HUD shows the final answer and a generic thinking spinner during tool
loops. It does **not** visualize intermediate steps: which tool was called,
what arguments, what came back. The full log lives in
`data/dialog_log.jsonl` (~163 KB at time of writing) but it is post-hoc.

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

Thread safety: handlers use `asyncio.run_coroutine_threadsafe` to bridge
non-async producers. On reconnect, the server replays sticky state (last
mode, last profile).

## Voice I/O

### STT

`voice/stt.py`, ~89 LOC. faster-whisper, model `small`, language `uk`.
Auto-detects CUDA. Uses float16 on GPU, falls back to int8 on CPU.

### TTS

`voice/tts.py`, ~277 LOC.

- Primary: Edge TTS (Microsoft) with Ukrainian Neural voices (`Ostap`,
  `Polina`). Online, free, very natural. Requires network.
- Fallback: Piper with the `uk_UA-lada-x_low` model. Fully offline. Robotic
  but functional.

The TTS module also extracts an RMS amplitude envelope to drive the HUD's
voice visualization during playback.

### Wake-word (disabled)

`voice/listener.py` integrates Porcupine for wake-word detection. The
required API key is not set, so `voice.enabled` is `false` in the config
by default. Wake-up uses the clap detector instead.

### Clap detector

`core/clap_detector.py` listens for a sharp clap on the microphone and
triggers activation. Simple amplitude threshold with a debounce. Good
enough for a personal device, not the right answer for production.

## Obsidian integration

`mcp_servers/obsidian_server.py`. Read-focused. Tools exposed:

- `vault_read(path)` — reads a markdown file
- `vault_list(path)` — lists a folder
- `vault_search(query)` — substring search
- `vault_daily_append(text)` — writes to today's daily note

The write tool exists but is blocked on practice by:

- A **privacy window**: writes are rejected when the screen is on (in
  `core/agent.py`, around line 1125).
- A **secret scanner**: writes containing patterns like `sk-ant-`, `sk-`,
  `AIza`, `ghp_`, `gsk_`, `tvly-` are rejected.

The vault paths the agent reads from:

- `vault/daily/` — daily notes for context
- `vault/projects/` — ongoing work
- `vault/decisions/` — decision history (written by the router on auth
  failures, only)
- `vault/learned/` — weekly digests
- `vault/claude-memory` — symlink to the persistent Claude memory directory

A separate watchdog daemon (`core/vault_sync.py`) syncs the vault to a
private git remote when enabled. Auto-push is `false` in config by default.

## Telegram bot

`core/telegram_listener.py`, ~1150 LOC. Owner-gated: an `OWNER_ID` constant
filters every callback. All non-owner messages are silently ignored.

Capabilities:

- Text in private chat → `agent.respond()`
- Voice OGG → Whisper → `agent.respond()`
- Group topic routing: topic IDs configured in `config.toml`
- Quick-menu reply keyboard (8 rows of shortcut buttons)
- `setChatMenuButton` — the "/" button in the text field shows only `/menu`

### Quick-menu buttons (as of 2026-06)

| Button | What it does |
|---|---|
| Показники | psutil snapshot: CPU, RAM, disk, battery, temperatures |
| Нотатки | List saved notes; inline ✅ (done) and 🗑 (delete) per note |
| Нагадування | List active reminders; inline 🗑 + `systemctl stop` per timer |
| Spotify | Inline transport: ⏮ ⏯ ⏭, volume ±10, current track. Auto-launches Spotify if not running |
| Таймер | ForceReply → minutes → `systemd-run --on-active` + desktop notify + TG message |
| Погода | wttr.in one-liner for the configured city |
| Буфер | ForceReply → read or write clipboard via xclip |
| Apps | Inline submenu: 12 desktop apps, launched via `app_launch()` |

### Spotify control

Spotify is controlled via D-Bus directly (`dbus-send` to
`org.mpris.MediaPlayer2.spotify`), without the Spotify Web API or OAuth.
This covers play/pause/next/previous and volume adjustment. If the
`spotify` process is not running, the bot launches it first via
`app_launch()` and then opens the transport keyboard.

The bot is opt-in (`tg.enabled = false` by default in config).

## MCP servers

There are 24 MCP servers under `mcp_servers/`. This is a large surface area
and it is acknowledged. Some are heavily used, some rarely. Below is the
function list. The exact tool counts per server may vary slightly.

| Server | Purpose |
|---|---|
| `obsidian_server.py` | Read-focused vault access |
| `vision_server.py` | Screen capture + describe through Ollama |
| (internal server) | Parallel internal pipeline (out of public scope) |
| `shell_server.py` | Shell command execution with allowlist |
| `git_server.py` | Local git operations |
| `notes_server.py` | Personal notes I/O |
| `reminders_server.py` | Time-based reminders |
| `web_search_server.py` | DuckDuckGo search |
| `spotify_server.py` | Spotify control (optional, OAuth) |
| `app_server.py` | Launch desktop apps |
| `system_server.py` | System info, brightness, audio profile |
| `file_server.py` | File read/write within allowed roots |
| ... | Plus ~12 more, smaller in scope |

If the project gets pruned, the MCP layer is the first place I'd start.

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

Missing from `requirements.txt` but used in code (a real bug to fix):

- `python-telegram-bot`
- `websockets`
- `porcupine`, `pyaudio` or `sounddevice`
- `faster-whisper`, `piper-tts`

## Failure modes

These are real and recurring. Each one cost real debug time.

| Failure | Cause | Mitigation |
|---|---|---|
| Fake tool calls | Claude emits a markdown ` ```shell_exec ` block instead of `tool_use` and then claims it ran | regex sanitizer in router; not complete |
| Local LLM latency >15s | GPU contention with a parallel internal pipeline | circuit breaker opens for 10 min |
| Ollama Xid 31 | post-suspend GPU MMU fault on NVIDIA driver | `modprobe nvidia_uvm` + systemd `Restart=always` |
| PipeWire audio race | ALSA EBUSY with WirePlumber on raw hw devices | prefer virtual `pipewire`/`pulse` devices |
| Lid-close double suspend | logind triggers a second suspend in the first 1–2 min after resume | avoid lid-close within that window |
| Stale Gemini session | `1008 BidiGenerateContent` on the live audio socket | delete `gemini_session.json`, restart |

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
- **Vault writes → read-only on practice** (early 2026). The privacy
  window plus secret scanner made write paths fail more often than they
  succeeded. Rather than fight it, the vault became a read source. A
  separate sync daemon handles writes from outside the agent.

## Testing

Three test files in `tests/`:

- `test_router_smart.py` — provider ordering, simple/complex heuristics
- `test_memory_v2.py` — facts, episodes, graph, decay
- `test_screen_context.py` — minimal coverage

There are no integration tests. There are no end-to-end voice tests. This
is a known gap. The unit tests catch the regressions I have actually hit;
the integration tests will land when an integration regression hits and
the lesson is paid in debug time.

## What the code is not

For completeness:

- Not a framework. There is no plugin system, no third-party extension
  contract.
- Not a research project. There are no novel ML contributions.
- Not multi-user. There is one `OWNER_ID` and one local profile.
- Not portable today. Hardcoded paths to a user-specific external disk
  mount exist in several files and would need to be parameterized before
  someone else could run this unchanged.
