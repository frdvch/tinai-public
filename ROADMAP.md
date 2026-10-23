# ROADMAP

Plans, status, and what comes next. Updated regularly — a stale roadmap looks abandoned.

---

## Done

- [x] 4-tier LLM routing (local → Groq → Haiku → Sonnet) — two tiers live under `local_only`, see ARCHITECTURE
- [x] ChromaDB semantic memory with exponential decay
- [x] Knowledge graph (networkx, nightly consolidation)
- [x] 24 MCP servers (web, shell, git, notes, reminders, vision, apps, windows, Obsidian, Spotify, bank statements, network, and more)
- [x] Whisper STT (local, CUDA float16 / CPU int8 fallback)
- [x] Telegram bot — mobile access to the full agent
- [x] Telegram menu as a persistent keyboard tree (apps, running apps, work schedule, finance, notes, reminders, Spotify D-Bus, utilities, metrics)
- [x] Circuit breaker for local LLM (2 failures or >15s → 10 min cloud fallback)
- [x] Local-only mode (`local_only = true` in config — no cloud API calls)
- [x] Local model on `qwen3:4b-instruct` — non-thinking variant, fits in 4 GB VRAM
- [x] Vault semantic search via Telegram — nomic-embed-text embeddings, ChromaDB index over Obsidian notes, `vault_recall` tool
- [x] README.md polish — routing order, Telegram menu, operational security note
- [x] Night mode — removed 2026-05-09 (overhead exceeded value)
- [x] Public repo with ARCHITECTURE.md, ROADMAP.md, blog
- [x] HUD redesign — frameless floating circle UI, status-only (chat input and transcript panel removed entirely, not just trimmed)
- [x] Local model "silent reasoning" bug fixed — model was looping in a hidden chain-of-thought step before every response, sometimes never converging
- [x] Meeting mode — suppresses proactive nudges during calls, optional auto-expiry on a timer
- [x] Telegram confirmation step before state-changing actions — profile switches, app closes, git push, deleting a reminder now require an explicit confirm tap, closing a "replies with text instead of doing it" failure mode
- [x] Cloud tier moved off the API wallet onto an OAuth subscription (`claude -p` as transport, API key stripped from the subprocess)
- [x] Read-only bank statement connector + spending analysis, categories from merchant type codes rather than from a model
- [x] Manual receipt tracker removed — used once in six months, replaced by the above
- [x] Rotating shift schedule: week/month views, editable from Telegram, reminder with acknowledge button before each shift
- [x] One source of truth for system metrics — averaged and peak temperatures with named sensors, fans, GPU — plus a persistent temperature log and a night summary in the morning briefing
- [x] Two rounds of deliberate bug-hunting with security fixes: path validation on vault reads, origin allowlist on the HUD websocket, per-action confirm tokens, secrets moved off NTFS to a restricted location
- [x] Persistent Telegram menu state — survives a service restart, with a reverse index so a tap never leaks through to the LLM
- [x] Test suite grown to 371 tests across 28 files, including a regression guard on the security fixes
- [x] Cloud vision path on photos removed — it spent real money while `local_only = true` promised it would not

---

## Planned

- [ ] Architecture diagram — Mermaid or SVG showing routing and data flow
- [ ] Test runner config — no `pytest.ini`, no `conftest.py`; the suite runs by path
- [ ] Fix `requirements.txt` — several imported packages are still unlisted (see ARCHITECTURE)
- [ ] Blog post: why tiers at all instead of just calling the biggest model
- [ ] Blog post: building local AI on RTX 3050 4GB — what fits and what doesn't
- [ ] Blog post: working with Claude Code as a solo developer
- [ ] Demo video or GIF — HUD + Telegram interaction, the single highest-impact missing asset

---

## On paper, not in code

- [ ] Reasoning visibility in HUD — step-by-step tool call view during agent loops

---

## Out of scope (deliberate)

- No multi-user support
- No cloud deployment of the full stack
- No mobile app
- No commercial offering
- No career-ops module — vault tracking for job postings and interview prep. Deliberately dropped, not pending

---

## Hardware upgrade horizon

Postponed, and the candidate is open. The 4 GB VRAM ceiling is the single
biggest constraint on local model quality, and for this workload more VRAM
on an older card beats fewer gigabytes on a newer one — which makes the
obvious new-card answer the wrong one. Subject to budget, and currently
behind stability work.

---

## Blog post backlog

Posts land in [`blog/en/`](blog/en/) and [`blog/uk/`](blog/uk/) in parallel.

- [x] Why I built tinAI instead of using ChatGPT
- [x] Replacing the local model: Gemma to Qwen after an 8-hour outage
- [x] How exponential decay memory works — and why I chose it
- [ ] What I learned building 24 MCP integrations
- [x] Removing the night mode: when good ideas are not worth the cost
- [ ] Working with Claude Code: the new shape of solo engineering
- [ ] The local model that silently reasoned itself in circles — a quiet Ollama bug and what it explained in hindsight
- [x] Fake Tool Calls: How My AI Assistant Learned to Lie About Doing Things
- [x] Deleting the feature I used once: a manual expense tracker and the bank API that replaced it
