# hackernoon-pou-drafts

Article drafts and cover images for a set of "Proof of Usefulness" (PoU)
submissions to [HackerNoon](https://hackernoon.com/), each tagged
`#proof-of-usefulness`.

This repository holds the source material — the polished Markdown articles and
their matching featured-image covers — used to publish a batch of write-ups
about small, single-purpose developer tools for working with LLMs and AI agents.

## What's in here

Each tool/project has a Markdown article describing the problem it solves and a
matching cover image under `covers/`.

| Draft | Cover | Topic |
|-------|-------|-------|
| `agentcast.md` | `covers/agentcast.png` | Validate-and-retry loop for structured LLM JSON output |
| `agentguard.md` | `covers/agentguard.png` | Network egress firewall for AI agents |
| `agentmemory.md` | `covers/agentmemory.png` | Memory for AI agents |
| `agentsnap.md` | `covers/agentsnap.png` | Snapshot tests for AI agents |
| `agenttrace.md` | `covers/agenttrace.png` | Cost and latency tracking for agent runs |
| `agentvet.md` | `covers/agentvet.png` | Validate tool args before execution |
| `bedrock-kit.md` | `covers/bedrock-kit.png` | Toolkit for Amazon Bedrock |
| `cachebench.md` | `covers/cachebench.png` | Detect lost prompt-cache hits before the bill lands |
| `driftvane.md` | `covers/driftvane.png` | Drift detection |

Additional long-form drafts and notes:

- `google-io-2026-writing.md`
- `hermes-agent-stack.md`
- `hn-algolia-gemini-search-agent.md`
- `hn-foundational-tech-llm-json-repair.md`
- `hn-neo4j-driftvane.md`

## Publishing

`PUBLISH.md` is the step-by-step runbook for submitting these drafts to
HackerNoon: which editor URL maps to which `.md` file and `covers/*.png`, how to
create a draft from a PoU report, and the editor fields (title, meta
description, TL;DR, tags, featured image) to fill in before submitting a story
for review.

## Repository layout

```
.
├── *.md            # article drafts (one per project) + long-form notes
├── covers/         # featured cover images (PNG) matching each draft
└── PUBLISH.md      # runbook for submitting drafts to HackerNoon
```

## Conventions

- Each publishable draft starts with a single `#` H1 line that doubles as the
  article title — drop the leading `#` when pasting the title into the editor.
- Every publishable draft maps to a `covers/<project>.png` of the same base
  name.
- Drafts intended for the contest include the `#proof-of-usefulness` tag.
