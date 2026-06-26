# ROADMAP

Plans, status, and what comes next. Updated regularly — a stale roadmap looks abandoned.

---

## Done

- [x] 4-tier LLM routing (local → Groq → Haiku → Sonnet)
- [x] ChromaDB semantic memory with exponential decay
- [x] Knowledge graph (networkx, nightly consolidation)
- [x] 25 MCP servers (web, files, shell, notes, reminders, vision, apps, Obsidian, Spotify, vault search, and more)
- [x] PyQt6 floating HUD (always-on-top, translucent, stats bar)
- [x] Whisper STT (local, CUDA float16 / CPU int8 fallback)
- [x] Telegram bot — mobile access to the full agent
- [x] Telegram quick-menu keyboard (Spotify D-Bus, system metrics, notes, reminders, timer, weather, clipboard)
- [x] Circuit breaker for local LLM (2 failures or >15s → 10 min cloud fallback)
- [x] Local-only mode (`local_only = true` in config — no cloud API calls)
- [x] Local model upgraded to `qwen3:4b-q8_0` (8-bit, 4.4 GB, fully in VRAM)
- [x] Vault semantic search via Telegram — nomic-embed-text embeddings, ChromaDB index over Obsidian notes, `vault_recall` tool
- [x] Night mode — removed 2026-05-09 (overhead exceeded value)
- [x] Public repo with ARCHITECTURE.md, ROADMAP.md, blog

---

## In progress

- [ ] README.md polish — clearer intro, routing explainer, "why I built this", status badge

---

## Planned

- [ ] Architecture diagram — Mermaid or SVG showing 4-tier routing and data flow
- [ ] Blog post: why 4 tiers instead of just calling GPT-4
- [ ] Blog post: building local AI on RTX 3050 4GB — what fits and what doesn't
- [ ] Blog post: working with Claude Code as a solo developer
- [ ] HUD refactor — remove transcript panel, keep stats bar and profile toggle only
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

Candidate: RTX 5060 Ti 16 GB. Trigger: hitting the 4 GB VRAM ceiling with larger local models (Qwen 7B+ or local embeddings). Timeline: 3–6 months from mid-2026, subject to budget.

---

## Blog post backlog

Posts land in [`blog/en/`](blog/en/) and [`blog/uk/`](blog/uk/) in parallel.

- [ ] Why I built tinAI instead of using ChatGPT
- [ ] How exponential decay memory works — and why I chose it
- [ ] What I learned building 25 MCP integrations
- [ ] Removing the night mode: when good ideas are not worth the cost
- [ ] Working with Claude Code: the new shape of solo engineering
