# Vault semantic search: from hallucinated memory to real memory

*2026-06-26*

---

*This post is part of a series on the technical decisions behind tinAI. The previous post covered the exponential decay memory system. This one is about a different kind of memory: the Obsidian vault, and what it took to make the assistant actually search it instead of guessing.*

---

Before this week, asking tinAI "what did we decide about X?" was a coin flip. Sometimes the answer came from genuine memory. Sometimes it came from the model's best guess about what a plausible answer would sound like. The assistant had no way to distinguish between the two, and neither did I, until the answer turned out to be wrong.

The vault has the actual history. Hundreds of notes, daily logs, project decisions, all written by hand or synced from conversations. The problem was that tinAI couldn't search it. It could read a file if you named it. It couldn't find the right file when you didn't know which one to ask for.

## What landed

The fix is a semantic search layer over the vault, built this week.

The pipeline: Ollama running `nomic-embed-text`, a small multilingual embedding model (~274 MB), chunks the vault into 655 pieces across 107 files. Those chunks go into ChromaDB. When a question comes in through Telegram, the query gets embedded the same way, ChromaDB returns the top 5 chunks by cosine similarity, and those chunks get handed to qwen3 as grounding context for the actual answer.

The difference in behavior is the whole point. Before, "what did we decide about X" produced a fluent, confident, sometimes wrong answer. Now it produces an answer built from five real fragments of text that were actually written, with the model synthesizing rather than inventing.

## Why this matters more than it sounds

It's tempting to read this as "added RAG to a chatbot," which undersells it. The actual shift is architectural: the vault went from being a passive archive — something a human reads — to being the system's source of truth for anything that happened before right now.

That has a consequence for how the assistant is allowed to answer. The rule going forward: tinAI does not answer questions about past decisions, project state, or prior conversations from its own training or context window. It answers from the vault, or it says it doesn't know. That's a stricter standard than most assistants hold themselves to, and it only became enforceable once the search actually worked.

## What's still rough

Reindexing the vault is manual right now, a script you run by hand. Anything written since the last index isn't searchable until someone remembers to run it. A scheduled reindex is the obvious next step.

The embedding model is multilingual, which matters since the vault is a mix of English and Ukrainian, but multilingual embeddings are a compromise — language-specific models would likely retrieve better. Worth revisiting if recall quality becomes a problem.

---

*tinAI is a local-first AI assistant running on an ASUS ROG laptop with an RTX 3050. Architecture and decisions are documented at [github.com/frdvch/tinai-public](https://github.com/frdvch/tinai-public).*
