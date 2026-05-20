# I shipped driftvane because my RAG was rotting and I had no way to tell

Tags: #ai-agents #rag #drift-detection #open-source #proof-of-usefulness #python #observability #ml-ops

## The silent failure mode

RAG quality decays without making a sound.

You swap the embedding model for a newer one. Recall shifts in ways your eval set never sees. The reranker version bumps on the vendor side and your top-k stops looking like it did last week. A chunk of your corpus gets reindexed by a teammate at 2am. The day your RAG starts hallucinating, nobody knows, because the only thing that would catch it is a full eval pass and that costs real money to run on every request.

So you run eval nightly, or weekly, or whenever someone complains in Slack. By the time the complaint lands, the bad answers have been going out for a week.

I got tired of this loop. So I wrote driftvane.

## What it is

driftvane is a small Python library. No server. No UI. No telemetry phoning home. It is a set of composable Detector classes you wire into your stack to watch for drift across the parts of a RAG pipeline that actually move.

```bash
pip install driftvane
```

Four detectors in v0.1.0:

- **EmbeddingDrift**: feed it two `(n, d)` arrays, get an MMD score with an RBF kernel.
- **RetrievalDrift**: feed it paired top-k id lists, get Jaccard at k and Rank-Biased Overlap.
- **ResponseDrift**: feed it intent / context / answer triples, get an answer-grounding shift score.
- **LatencyDrift**: feed it two arrays of latencies, get a Kolmogorov-Smirnov statistic.

You compose them into one `DriftReport`. That is the whole shape of the library.

## How it works

The pattern is the same for every detector. You hold on to a reference slice (last week, last release, last known-good window). You sample a current slice from live traffic. You call `.compute(ref, cur)` and you get a signal back. You stuff the signals into a `DriftReport`.

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
    ResponseDrift(threshold=0.15).compute(ref_triples, cur_triples),
    LatencyDrift(p_threshold=0.01).compute(ref_latencies, cur_latencies),
])

if report.any_drifted():
    print(report.to_pandas())
```

That is the whole loop. Run it on a slice of yesterday's traffic versus today's. Run it as a nightly job. Run it on a sample inside your request handler if you are brave.

There is also a gate pattern, for when you want CI to fail on bad numbers:

```python
from driftvane import DriftAlert
import sys

try:
    report.alert_if({"retrieval_jaccard_at_10": 0.2})
except DriftAlert as e:
    sys.exit(f"drift gate failed: {e}")
```

Wire that into your deploy pipeline and a bad embedding-model swap stops the release instead of leaking into prod.

## Why I think it matters

Agents that depend on retrieval are the most fragile part of the modern LLM stack and the part nobody is watching. We watch latency, we watch error codes, we watch token cost. We do not watch whether the retrieval layer still surfaces what it surfaced last week.

The reason we do not watch is not that we do not care. It is that the existing options are heavy. Vendor eval platforms want you to pipe everything through them. OTel-based observability needs a collector and a backend. Both are fine if you already run that stack. Neither helps if you just want a numpy array compared to another numpy array on a cron.

driftvane is the numpy-array-on-a-cron option. It runs in a Lambda. It runs in a notebook. It runs in a GitHub Action. It has a numpy core and that is most of what it depends on. Pandas is an optional extra for `to_pandas()`.

It sits on top of whatever RAG stack you already have. LangChain, LlamaIndex, your own hand-rolled retriever, a Bedrock Knowledge Base, a Pinecone client. It does not care. You hand it arrays.

## What is missing

v0.1.0 is an alpha. I want to be honest about that.

The detectors are intentionally minimal. EmbeddingDrift is one MMD with one kernel. ResponseDrift is one grounding-shift signal. There is no server, no dashboard, no tabular feature drift, no live OTel ingestion, no root-cause analysis, no auto-retraining trigger. The public API may change before v1.0.

That is on purpose. The point of v0.1.0 is to be small enough that you can read the source in one sitting, fork a Detector, and write your own. If you want a Wasserstein-based embedding test instead of MMD, that is a fifty-line class. If you want a custom grounding metric for your domain, the Detector interface is one method.

The composability is the product. The four built-in detectors are the proof that the interface holds.

## Try it

- GitHub: https://github.com/MukundaKatta/driftvane
- PyPI: `pip install driftvane`
- PoU report: https://proofofusefulness.com/reports/driftvane-composable-rag-and-agent-drift-detectors (scored 53.19)
- License: MIT

If you run RAG in production and you cannot tell me what your retrieval Jaccard-at-10 was last Tuesday, you have the problem driftvane is built for. Wire it in. Run it on a slice. See what the numbers say.

I would rather hear from people whose drift gate failed than from people whose users noticed first.
