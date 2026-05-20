---
title: "agentmemory: pull-model episodic memory for AI agents"
tags: [ai-agents, memory, open-source, proof-of-usefulness, llm, developer-tools]
---

# agentmemory: pull-model episodic memory for AI agents

Most agent memory libraries push everything they have into the context window. The agent boots, the library shoves a transcript or a summary blob into the system prompt, and the token budget is half gone before the user has typed anything. Most of that material is irrelevant to the turn the agent is about to handle.

I got tired of paying for tokens I did not ask for. So I built agentmemory the other way around. The agent pulls memory by intent. Nothing is injected unless I call for it.

This is my entry for the Proof of Usefulness Hackathon. PoU score 48.

## What it is

agentmemory is a small Node library. Three classes, zero runtime dependencies, under 500 lines of source. Real deletes, no tombstones, no background consolidation jobs that bake memories into something you cannot remove later.

Install:

```bash
npm install @mukundakatta/agentmemory
```

Node 20 or newer. Pure ESM.

The core recall flow is two calls:

```js
import { EpisodicStore } from "@mukundakatta/agentmemory";

const store = new EpisodicStore({
  embedder: async (text) => myEmbedder(text),
});

await store.append({
  sessionId: "user-42",
  kind: "user_message",
  text: "I prefer Postgres for the new project",
});

const hits = await store.retrieve("which database should I use", {
  sessionId: "user-42",
  topK: 5,
});
```

That is it for the retrieval side. You append events as they happen. When the agent needs memory, it asks for memory.

If you do not pass an embedder, the store falls back to keyword overlap. Good enough for tests and small agents.

## How it works

The store is an append-only event log. Each event has a session id, a kind, and text. At write time the embedder runs and the vector is kept next to the row. Retrieval is a cosine pass over the embeddings, filtered by session, time, and kind.

The interesting piece is the second class, the on-demand summarizer.

```js
import { OnDemandSummarizer } from "@mukundakatta/agentmemory";

const summarizer = new OnDemandSummarizer({
  llm: async (prompt) => callMyLLM(prompt),
  maxTokens: 300,
});

const events = await store.retrieve("pick a database", { topK: 5 });
const { summary, trace } = await summarizer.summarize(events, "pick a database");

console.log("Summary:", summary);
console.log("Built from event ids:", trace.eventIds);
console.log("Prompt sent to LLM:", trace.prompt);
```

You bring your own LLM. The summarizer takes the retrieved events plus the intent string and asks the LLM for a short summary. The trace object hands back the exact event ids and the exact prompt used. You can show that to the user before it goes into the model context. You can log it to an audit trail. Nothing silent.

That is the pull model. The agent decides when to recall. The library returns a summary that shows its work.

The third class is a drift watcher. It tracks retrieval scores across calls. If yesterday's queries used to return high scores and today they are returning nothing, you get a signal instead of a silent regression. Small piece, but it matters once you run this in production for a few weeks.

For storage, the default is in-memory. There is a Postgres adapter behind the same interface. Real deletes via DELETE, no soft-delete columns, no derived tables that survive after the source row is gone. If a user asks you to forget something, you can.

## Why it matters

Most agent memory today is vector search over a transcript. You dump every turn into a store, embed it, and retrieve nearest neighbors when the user types something new. That is not how human memory works, and it is not how agent memory should work either.

Humans recall by intent. You do not load your entire past into working memory every time someone asks you a question. You decide what the question is about, then you reach for the relevant thing. Sometimes you fail and reach for the wrong thing. That is fine. The failure is visible to you.

The push model hides the failure. You never see what the library decided to inject. You see the agent's output and it feels off, but the chain back to the bad retrieval is not in front of you. The pull model puts the retrieval call in the agent's main loop, returns a trace with the event ids and the prompt, and makes the failure mode legible.

The other half of why this matters is deletion. If you run a background consolidation pass that bakes episodic memories into a semantic summary, you cannot really forget anything. The summary stays even if you delete the source events. agentmemory does no background work. Append, retrieve, delete. The deletes are real.

## What is missing

This is v0.1.0. Honest list:

- The default store is in-memory. Fine for tests, not for a production agent running across restarts. Use the Postgres adapter or write your own backend behind the same interface.
- The keyword fallback is basic. If you skip the embedder you will hit recall limits fast.
- No built-in eviction policy beyond delete-by-age and delete-by-session. You can build retention on top, but the library does not opine.
- No multi-tenant isolation beyond session id. If you run shared infra you need to add that layer yourself.
- The drift watcher is simple. Mean-score drop over a sliding window. For real drift math I have a sibling library called ragdrift, but I did not pull it in as a dependency here. Zero runtime deps was the rule.

None of that is hidden. The whole library is short enough to read in one sitting.

## Try it

Repo: https://github.com/MukundaKatta/agentmemory

Install:

```bash
npm install @mukundakatta/agentmemory
```

License: MIT.

If you want a runnable end-to-end demo, the examples folder has one that wires the store and the summarizer to a real LLM, runs two sessions, retrieves across them, prints the summary before it goes into context, and does a real delete at the end. That is the shape of the thing.

The pull model is not a clever optimization. It is a different default for how agents should reach for what they remember. Less magic, more legible, real deletes. That is the trade.

#ai-agents #memory #open-source #proof-of-usefulness #llm #developer-tools
