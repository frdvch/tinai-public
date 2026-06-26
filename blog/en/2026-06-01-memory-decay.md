# How exponential decay memory works — and why I chose it

*2026-06-01*

---

*This post is part of a series on the technical decisions behind tinAI. This one covers the memory system: how it stores things, how it forgets them, and why forgetting is a feature.*

---

Most AI assistants either have no persistent memory at all, or they accumulate everything forever. Neither is right for a personal assistant you use every day.

No memory means repeating yourself constantly. Infinite accumulation means the context gets noisy over time — old irrelevant facts compete with recent ones, and the assistant starts surfacing things that stopped mattering months ago.

tinAI's memory system does something different. It forgets.

## Three layers

Memory in tinAI has three layers.

The first is a ChromaDB vector database with three collections: facts (preferences, habits, work patterns extracted automatically from conversations), episodes (daily summarized interactions), and reflections (output of the nightly consolidation process). Search runs on cosine similarity against the embeddings.

The second is a knowledge graph built with networkx. Nodes are entities — people, projects, tools. Edges are relationships between them. The graph is built incrementally by the nightly consolidation job, not updated in real time from raw chat.

The third layer is the forgetting curve. This is the part that makes the system behave like memory rather than a log.

## The decay function

The math is simple:

```
weight = exp(-Δdays / halflife)
```

Halflife is set to 30 days. Threshold is 0.1. Anything that decays below 0.1 gets dropped on the next nightly consolidation run.

What this means in practice: a fact from yesterday has a weight close to 1.0. The same fact from 3 months ago has a weight around 0.05 and gets dropped. A fact that gets referenced regularly resets its decay clock and stays relevant.

The decay runs once a night at 02:00. If the machine was suspended at that time, it catches up on next wake.

## What the nightly consolidation does

The consolidation job runs five steps:

1. Aggregate the day's raw facts into episodes.
2. Extract entities and relations from new episodes into the knowledge graph.
3. Apply decay to ChromaDB facts and drop anything below threshold.
4. Write a summary to `data/last_consolidation.json`.
5. On Sundays, write a weekly digest into the Obsidian vault.

The graph edges don't decay yet. That's a known gap — the graph grows indefinitely while the vector store stays clean. It's on the list.

## Why this approach

The alternative I considered was a sliding window: keep the last N facts, drop the rest. Simpler to implement, but it throws away things by age regardless of relevance. A fact from 3 months ago that's still being referenced regularly would get dropped at the same rate as something that was mentioned once and never again.

Exponential decay weights by time but implicitly rewards relevance through rereferencing. It's not perfect — there's no explicit relevance signal, just time — but it's a better model of how human memory actually works than a flat window.

The halflife of 30 days was chosen empirically. Long enough that things don't disappear after a week. Short enough that genuinely stale facts don't accumulate for years. It's a tunable parameter; 30 days felt right after the first month of use.

## What this looks like in practice

After several months of daily use, the facts collection stays manageable. The assistant doesn't surface conversations from January when you're asking about something in June. Recent context is weighted higher automatically without manual curation.

The knowledge graph is less clean — it needs the decay implementation. But even without it, the graph is useful for entity relationships that don't really go stale: the assistant knows what tools are part of which projects, what people are connected to what work.

---

*tinAI is a local-first AI assistant running on an ASUS ROG laptop with an RTX 3050. Architecture and decisions are documented at [github.com/frdvch/tinai-public](https://github.com/frdvch/tinai-public).*
