# ROADMAP

Plans, status, and what comes next. Updated regularly — a stale roadmap looks abandoned.

---

## Done

- [x] 4-tier LLM routing (local → Groq → Haiku → Sonnet)
- [x] ChromaDB semantic memory with exponential decay
- [x] Knowledge graph (networkx, nightly consolidation)
- [x] 25 MCP servers (web, files, shell, notes, reminders, vision, apps, Obsidian, Spotify, vault search, and more)
- [x] Whisper STT (local, CUDA float16 / CPU int8 fallback)
- [x] Telegram bot — mobile access to the full agent
- [x] Telegram quick-menu keyboard (Spotify D-Bus, system metrics, notes, reminders, timer, weather, clipboard)
- [x] Circuit breaker for local LLM (2 failures or >15s → 10 min cloud fallback)
- [x] Local-only mode (`local_only = true` in config — no cloud API calls)
- [x] Local model upgraded to `qwen3:4b-q8_0` (8-bit, 4.4 GB, fully in VRAM)
- [x] Vault semantic search via Telegram — nomic-embed-text embeddings, ChromaDB index over Obsidian notes, `vault_recall` tool
- [x] README.md polish — routing order, Telegram quick-menu, operational security note, LinkedIn
- [x] Night mode — removed 2026-05-09 (overhead exceeded value)
- [x] Public repo with ARCHITECTURE.md, ROADMAP.md, blog
- [x] HUD redesign — frameless floating circle UI, voice-only (chat/transcript panel removed entirely, not just trimmed)
- [x] Local model "silent reasoning" bug fixed — model was looping in a hidden chain-of-thought step before every response, sometimes never converging
- [x] Meeting mode — suppresses proactive nudges during calls, optional auto-expiry on a timer
- [x] Telegram confirmation step before state-changing actions — profile switches, app closes, git push, deleting a reminder now require an explicit confirm tap, closing a "replies with text instead of doing it" failure mode

---

## Planned

- [ ] Architecture diagram — Mermaid or SVG showing 4-tier routing and data flow
- [ ] Blog post: why 4 tiers instead of just calling GPT-4
- [ ] Blog post: building local AI on RTX 3050 4GB — what fits and what doesn't
- [ ] Blog post: working with Claude Code as a solo developer
- [ ] Demo video or GIF — HUD + Telegram interaction, the single highest-impact missing asset

---

## On paper, not in code

- [ ] Career-ops module — vault tracking for job postings and interview prep (`core/personal_journal.py` is an empty stub)
- [ ] Reasoning visibility in HUD — step-by-step tool call view during agent loops

---

## Out of scope (deliberate)

- No multi-user support
- No cloud deployment of the full stack
- No mobile app
- No commercial offering

---

## Hardware upgrade horizon

Candidate: RTX 5060 Ti 16 GB. Trigger: hitting the 4 GB VRAM ceiling with larger local models (Qwen 7B+ or local embeddings). Subject to budget.

---

## Blog post backlog

Posts land in [`blog/en/`](blog/en/) and [`blog/uk/`](blog/uk/) in parallel.

- [x] Why I built tinAI instead of using ChatGPT
- [x] Replacing the local model: Gemma to Qwen after an 8-hour outage
- [x] How exponential decay memory works — and why I chose it
- [ ] What I learned building 25 MCP integrations
- [x] Removing the night mode: when good ideas are not worth the cost
- [ ] Working with Claude Code: the new shape of solo engineering
- [ ] The local model that silently reasoned itself in circles — a quiet Ollama bug and what it explained in hindsight
- [x] Fake Tool Calls: How My AI Assistant Learned to Lie About Doing Things
