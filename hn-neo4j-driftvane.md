# Your GraphRAG Pipeline Is Drifting and You Probably Have No Idea

GraphRAG looks beautiful on day one.

You wire Neo4j into your retriever, the answers come back grounded, citations point at real nodes, and your stakeholders nod. Six weeks later, recall is down something like a third and nobody on the team knows. The pipeline still returns answers. They are just worse ones.

Here is what usually happened in those six weeks:

- The embedding model got bumped by a point release and the vector space shifted under you.
- A hot subgraph got rewritten by a content team. New edge types. Old ones gone.
- The reranker quietly drifted because its training distribution no longer matches your traffic.
- A schema migration added a node label that the Cypher query never filters on.

Any one of those is invisible to a green dashboard. Latency is fine. Errors are zero. The model still produces fluent text. The only thing that changed is that the retrieval layer is now feeding it slightly worse context, and the answers are slightly more wrong, and nobody is paging anybody.

I got tired of this story. So I wrote a small Python library called **driftvane** to catch the drift before the next quarterly review catches it for you.

## What driftvane is

driftvane is a composable drift detector for RAG and agent systems. No server. No telemetry backend. You install it, you run it on a slice of traffic, and you get back a report.

```bash
pip install driftvane
```

It ships with four detectors:

1. **EmbeddingDrift**. Maximum Mean Discrepancy with an RBF kernel over two `(n, d)` embedding matrices. Catches the case where your embedding model or your input distribution moved.
2. **RetrievalDrift**. Paired top-k ID lists, scored as `1 - mean Jaccard@k`. Catches the case where the retriever is returning a different set of documents or nodes for the same kind of query.
3. **ResponseDrift**. `(intent, context, answer)` triples, scored as grounding shift between the answer and the context. Catches the case where the model is leaning less on the retrieved context over time.
4. **LatencyDrift**. Two 1-D float arrays, scored as the Kolmogorov-Smirnov D statistic. Catches the case where p95 quietly doubled.

Each detector returns a `DriftSignal(name, value, threshold, drifted, metadata)`. You compose them into a `DriftReport` and gate on it.

## How it works

The whole point is that you do not need to instrument every call. You sample. You keep a reference window from a known-good period. You compare the current window to it on a cron.

```python
from driftvane import (
    DriftReport,
    EmbeddingDrift,
    RetrievalDrift,
    ResponseDrift,
    LatencyDrift,
)

report = DriftReport.from_signals([
    EmbeddingDrift(threshold=0.1).compute(ref_emb, cur_emb),
    RetrievalDrift(k=10, threshold=0.3).compute(ref_top_k, cur_top_k),
    ResponseDrift(threshold=0.2).compute(ref_triples, cur_triples),
    LatencyDrift(threshold=0.15).compute(ref_latency_ms, cur_latency_ms),
])

if report.any_drifted():
    print(report.to_pandas())

# Or gate hard in CI / on a schedule
report.alert_if({
    "embedding_mmd": 0.1,
    "retrieval_jaccard": 0.3,
})
```

`alert_if` raises a `DriftAlert` when any metric breaches its threshold. That is your pager hook. Wire it to Slack, to a GitHub issue, to whatever you already have. The library does not care.

## Why this matters for graph-powered retrieval

If your retrieval layer is a vector index, drift mostly shows up in the embedding signal. The math is well understood and the failure modes are roughly the same shape.

If your retrieval layer is a knowledge graph, drift looks different.

Neo4j's GraphRAG patterns are the right way to give an LLM structured context. Their docs on knowledge-graph context for LLMs are some of the clearest writing on the subject, and the pattern works. A graph gives you typed relationships, multi-hop reasoning, and citations that point at real entities instead of chunked PDF pages. That is a real upgrade over plain vector retrieval.

But the graph is also a moving target in ways a static vector store is not:

- **Schema evolves silently.** A new node label or a renamed property changes what your Cypher query can match. The query still runs. It just returns a different shape.
- **New edge types shift ranking.** If your retrieval scores edges by type frequency or by path length, a content team adding a new relationship type can move every score in the index.
- **Subgraph churn is uneven.** One community of nodes gets rewritten heavily. Most of the graph sits still. Aggregate metrics hide the local damage.
- **Embedding-on-graph compounds both.** When you embed node neighborhoods, a graph change shows up as an embedding change too, but the root cause is in the graph, not in the model.

This is where driftvane's composability is the point. The four built-in detectors give you the vector-side and latency-side signals out of the box. The `DriftReport.from_signals` primitive takes any object that looks like a `DriftSignal`. So you can plug in a graph-aware signal of your own next to the embedding signal and treat them the same way.

A graph-aware signal might be: Jaccard distance over the set of node labels touched per query, between the reference window and the current window. Or a KS test on path lengths. Or a count of queries that hit a node label that did not exist in the reference window. You compute it, you wrap it as a `DriftSignal`, you pass it into `from_signals` alongside the embedding and retrieval detectors, and the same alert gate fires on it.

The graph is the source of truth in GraphRAG. It deserves its own drift signal sitting next to the vector one, not buried under it.

## What is still missing

This is the honest part.

driftvane is v0.1. The four detectors in the box are vector-shaped and stats-shaped. There is no out-of-the-box `GraphSchemaDrift` detector that knows about Neo4j labels or relationship types. If you want that today, you write it as a custom detector that returns a `DriftSignal` with the right fields.

That is a real gap. It is also the most interesting place for the library to grow, and the composability primitive is already there so the gap is fillable without rewriting anything.

If you build a graph-side detector on top of this and it works, I want to see it. Open an issue, send a PR, or just post about it.

## Try it

- GitHub: [github.com/MukundaKatta/driftvane](https://github.com/MukundaKatta/driftvane)
- PyPI: `pip install driftvane`
- License: MIT

The whole library is small on purpose. Read the source, fork it, swap in your own detectors. The goal is to make drift a thing your pipeline notices on its own, not a thing your users notice for you six weeks later.

If you are running a GraphRAG stack on Neo4j and you do not have a drift gate today, add one. It does not have to be this library. It just has to exist. Silent decay is the failure mode that ships to production the most often and gets caught the least often, and graphs make it worse because the surface area of "what changed" is bigger than a vector index.

Catch it early. Page yourself, not your users.

---

*Tags: #ai #neo4j #graphrag #drift-detection #rag #python #open-source #foundational-tech #observability #knowledge-graph*
