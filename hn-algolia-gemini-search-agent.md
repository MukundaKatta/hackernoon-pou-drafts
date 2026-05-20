# A Small Gemini Agent + Managed Retrieval Beats a Giant RAG Pipeline for the Questions That Actually Matter

Tags: #ai #algolia #search #retrieval #elasticsearch #gemini #ai-agents #open-source

## The day "around 50 to 100" stopped being an answer

A support lead pinged me on a Tuesday afternoon. Customers were complaining about checkout. She wanted one thing:

> How many users hit error code 503 in the last hour, by region?

The team had a RAG pipeline sitting on top of the log archive. Pretty embeddings. Hybrid search. A reranker. The whole party.

The answer it gave back was:

> Around 50 to 100 users were affected, mostly in the US and EU.

That is not an answer. That is a vibe.

Counts are exact. Either 412 users got a 503 between 14:00 and 15:00, or they didn't. A retrieval system that summarizes its way past a number is the wrong tool for the question. So I stopped trying to make my RAG pipeline answer counting questions and built a small agent that asks the cluster directly.

That project is `gemini-search-agent`, and the rest of this post is what I learned shipping v0.1.0.

## What gemini-search-agent is

It is a small Python agent. Gemini sits in the middle. The Elastic MCP server sits on the side. The agent turns plain-English questions into Elasticsearch DSL, runs them, and prints the answer with the exact query and the doc IDs it used.

No vector store. No reranker. No embedding step. The cluster already knows how many 503s it saw. I just need the agent to ask it the right way.

A session looks like this:

```
> how many users hit error code 503 in the last hour, by region?

ANSWER
  US-East:  412
  EU-West:  198
  AP-South:  37
  total:    647

QUERY
  POST logs-prod-*/_search
  {
    "size": 0,
    "query": {
      "bool": {
        "filter": [
          { "term":  { "http.response.status_code": 503 } },
          { "range": { "@timestamp": { "gte": "now-1h" } } }
        ]
      }
    },
    "aggs": {
      "by_region": {
        "terms": { "field": "cloud.region", "size": 20 }
      }
    }
  }

CITATIONS
  index: logs-prod-2026.05.20
  hits scanned: 647
  sample doc ids: log-7f3a..., log-9c11..., log-bd02...
```

That output shape is the whole point. The ANSWER is a number the cluster returned. The QUERY is the exact DSL the agent built, so I can paste it into Kibana and reproduce. The CITATIONS prove the count came from real docs in a real index.

If the agent makes up a number, the QUERY block tells me where it lied.

## How it works

The agent has four tools wired through the official `@elastic/mcp-server-elasticsearch` MCP package:

- `list_indices` to see what is in the cluster
- `get_mappings` to learn the field schema before writing a query
- `search` to run the query
- `count_errors` for the common "how many errors in window X" path

A turn looks like this. Gemini gets the user question. It calls `list_indices`, picks `logs-prod-*`, calls `get_mappings` to confirm `http.response.status_code` is a keyword field, builds the DSL, calls `search`, then formats the labeled output.

The install is one screen:

```bash
git clone https://github.com/MukundaKatta/gemini-search-agent
cd gemini-search-agent
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT=your-project
export GOOGLE_GENAI_USE_VERTEXAI=true
export GOOGLE_CLOUD_LOCATION=us-central1

export ES_URL="https://your-cluster.es.region.gcp.elastic-cloud.com"
export ES_API_KEY="..."

PYTHONPATH=src streamlit run app/dashboard.py
```

That is it. No index build step. No reembedding job at 3am. The cluster you already have is the retrieval layer.

## Why managed retrieval is the part you should not roll yourself

The mistake I made for a year was treating retrieval as a thing I should build. Pull pgvector in. Pick a chunker. Argue about cosine vs dot product. Tune top-k. Add a reranker. Watch recall slowly rot as the corpus grows.

The boring truth is that Algolia and Elastic have spent more engineering on ranking, recall, and query latency than my side project ever will. They both ship:

- query parsers that handle typos, synonyms, and language quirks
- ranking that you can tune with real signals instead of guesses
- aggregations that return exact counts, not "around"
- latency budgets that hold up when the corpus is large
- a clear story for sharding, replicas, and failover

When I let the agent drive a managed retrieval layer, the agent's job shrinks. It picks a tool, writes a query in that tool's language, and reads the response. It does not have to be the search engine. It just has to know which search engine to talk to.

That is also why the same agent pattern works for both Elastic and Algolia. They cover different shapes of question well:

- Algolia is the right pick for product search, autocomplete, faceted browsing, and recommendations on a storefront. Sub-100ms responses, ranking strategies tuned for commerce, typo tolerance out of the box.
- Elastic is the right pick for logs, events, time-series, and high-cardinality aggregations. The DSL is built for the "how many, by what, in which window" questions that ops teams live in.

A small agent in front of both is just a router. "Did the user ask about a product catalog or about logs?" Pick the tool. Write the query. Cite the response.

I am building the Elastic side first because that was my Tuesday-afternoon fire. The Algolia adapter is the next thing on the list, and the agent shape does not change. New tool. Same labeled output. Same rule: if the engine returned 412, the answer is 412.

## What is missing in v0.1.0

I want to be honest about where this is.

- Only Elastic is wired today. The Algolia adapter is sketched, not shipped.
- There is no multi-cluster routing. One `ES_URL` at a time.
- The agent will write a bad query if the field mapping is unusual. It catches the error and retries once, but it does not learn across sessions yet.
- Auth is whatever Vertex AI and Elastic API keys give you. No per-user scoping inside the agent.
- The Streamlit dashboard is a developer surface, not a polished UI.

None of that blocks the core use case, which is: a small team that needs a real number from a real index without standing up a RAG stack.

## Try it

The repo is here: https://github.com/MukundaKatta/gemini-search-agent

Install steps are above. License is Apache 2.0. Open an issue if your cluster shape breaks the agent. I am especially interested in field-mapping cases where the agent's first query is wrong, because that is the loop I want to tighten next.

If you are sitting on top of Algolia and want to be the first person to wire that adapter, that PR has my name on it before you even send it.

The lesson I keep coming back to: the search engines already won. Let them rank and count. Put a small agent in front to translate human questions into their query language, and print the answer with the receipts attached.

"Around 50 to 100" is not an answer. 647 is.
