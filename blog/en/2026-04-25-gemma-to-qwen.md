# Gemma to Qwen: replacing the local model after an 8-hour outage

*2026-04-25*

---

*This post is part of a series on the technical decisions behind tinAI. The previous post covered why the project exists. This one is about what happens when your local model stops being reliable.*

---

tinAI runs a 4-tier routing stack: local model first, then Groq, then Claude Haiku, then Claude Sonnet. The local tier is the foundation. When it breaks, everything else gets more expensive.

For the first few months, the local model was Gemma 3 4B.

## What broke

Gemma 3 4B fit the constraint: 4 GB VRAM, quantized, runs fully in memory. On simple queries it was fine. Under load it was not.

The failure mode was latency. Under GPU contention — which happens when a parallel pipeline is running on the same machine — Gemma response times climbed to 30-150 seconds. Not a crash, not an error. Just slow, unpredictably, in a way the circuit breaker couldn't always catch cleanly.

The worst incident was an 8-hour GPU cascade. Something in the interaction between Gemma and the NVIDIA driver under sustained load caused the GPU to become progressively less responsive until a full restart was the only fix. Eight hours of the local tier being effectively dead, all traffic falling through to Claude.

After that I started looking for a replacement.

## Why Qwen

The requirement was simple: same footprint, better stability. Qwen 3 4B q8_0 fit both.

It runs at 4.4 GB in VRAM — just within the 4 GB ceiling at 8-bit quantization. The behavior under load is noticeably different. No cascading latency under GPU contention. The circuit breaker still exists and still triggers occasionally, but the failure mode is a clean timeout rather than a slow degradation.

Two other things mattered. Qwen has better Ukrainian language support than Gemma, which matters for a system built around Ukrainian voice input. And Qwen's tool calling is stronger — fewer cases where the model pretends to call a tool instead of actually calling one.

## What didn't change

The config section is still named `[llm.gemma]` and the function `_call_gemma` still exists in `core/router.py`. That refactor is on the list. The naming is wrong but the behavior is right — the function calls Qwen now regardless of what it's called.

This is the kind of debt that accumulates in personal projects. It's documented here so it's not a surprise when someone reads the code.

## The lesson

Local model stability is not just about whether the model fits in VRAM. It's about how the model behaves when the machine is doing other things at the same time. A model that runs clean in isolation can degrade badly when the GPU is contended.

Test under realistic load, not in a clean environment. The clean environment is not where your assistant runs.

---

*tinAI is a local-first AI assistant running on an ASUS ROG laptop with an RTX 3050. Architecture and decisions are documented at [github.com/frdvch/tinai-public](https://github.com/frdvch/tinai-public).*
