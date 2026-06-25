# ROADMAP

This document tracks open questions, planned work, and post candidates for the
[blog](blog/en/). It is intentionally short and honest about what is real code
versus what is on paper.

## Status snapshot (2026-06-26)

- `shadow.service` active and enabled (autostart on boot)
- `shadow-restart.timer` active and enabled
- Local model: `qwen3:4b-q8_0` (8-bit quantized, 4.4 GB, fully in VRAM)
- Cloud API temporarily disabled via `local_only = true` in config
- Telegram bot substantially expanded since May (see Architecture for details)

## Upcoming

- **Vault semantic search via Telegram.** Natural-language queries ("what
  did we decide about X?") will trigger a ChromaDB similarity search over
  indexed Obsidian vault files, with qwen3 synthesizing a response. No new
  collection yet — vault indexer not written.
- **HUD refactor.** Remove the text transcript and reminder panels from the
  overlay; keep only the system stats bar and profile toggle. Deferred
  because the current web-based HUD works well enough and the refactor is
  nontrivial.

## On paper, not in code

These are designed but not implemented. Mentioned for honesty.

- **Career-ops module.** Vault folders for tracking job postings and
  interview preparation. The Python side (`core/personal_journal.py`) is
  an empty stub.
- **Reasoning visibility in the HUD.** Currently only a thinking spinner
  appears during tool loops. A step-by-step view of tool calls is on the
  wishlist but not designed yet.

## Current branch — what's left

Items that need to land before the branch can be merged into `master`:

- Finish `core/meeting_time_parser.py` (currently untracked)
- Finish `scripts/wake_resume_starter.py` (currently untracked)
- Commit the 13 modified files (router, telegram, web HUD, WebSocket, vision
  server, face HUD assets, audio utils, AFK monitor, safe-restart script)
- Commit the deletion of `core/voice_loop.py`
- Clarify Haiku tier routing — the config defines `claude-haiku-4-5` but
  most queries still route to Sonnet in practice. Either document the
  intended trigger conditions or change the heuristic.
- Rename the legacy `[llm.gemma]` config section after the Gemma 3 4B → Qwen
  3 4B migration. The router function `_call_gemma` should become
  `_call_local_llm`. Cosmetic, but the inconsistency is misleading.
- Reconcile `ollama_model = "qwen2.5:7b"` (top-level) vs
  `[llm.gemma].model = "qwen3:4b"` (section). Currently the router reads
  the section value, so Qwen 3 4B is what actually runs.

## Hardware upgrade horizon

- Window: 3 to 6 months from 2026-05-16
- Candidate: RTX 5060 Ti 16GB
- Trigger: the career-ops module matures into real code and hits the 4GB
  VRAM ceiling with larger local models (Qwen 7B or 14B, or local
  embeddings)
- Not on the upgrade list: a server, a cluster, anything cloud-hosted

## Blog post candidates

Drafts will land in [blog/en/](blog/en/) (English) and [blog/uk/](blog/uk/)
(Ukrainian). Order is rough.

1. **Why I built tinAI instead of using ChatGPT.** Local-first as a
   response to seeing infrastructure collapse during war. Privacy as a
   default, not an upsell.
2. **Read-only contract: how I let my AI write without breaking trust.**
   The Obsidian integration ended up read-only on practice (privacy window
   plus secret scanner). Why that's a feature.
3. **Building local AI on RTX 3050 4GB: what fits and what doesn't.** A
   concrete look at Qwen 3 4B inference, Whisper STT under 4GB VRAM, and
   where the ceiling actually sits.
4. **Working with Claude Code: the new shape of solo engineering.** Honest
   description of the partnership: architectural decisions mine,
   implementation paths suggested by Claude, choices made on tradeoffs I
   could articulate. Not "I built this from scratch." Not "Claude built
   this."
5. **Removing the night mode: when good ideas are not worth the cost.**
   The night mode feature was removed on 2026-05-09 after multi-day debug
   overhead exceeded its value. A short post on knowing when to cut.

## Out of scope (deliberate)

- No multi-user support
- No web UI for remote use
- No mobile app
- No cloud deployment of the full stack
- No commercial offering, no SaaS, no donation buttons

If any of these change later, the roadmap will be updated.
