---
title: "What I shipped in 6 weeks after Google's Agent Development Kit landed"
published: false
description: "A builder's-eye view of the post-I/O Gemini stack. Seven small agents, one repeating pattern, and the rules I won't ship without."
tags: google-io-2026, gemini, vertex-ai, agent-development-kit, ai-agents, mcp, open-source
---

# What I shipped in 6 weeks after Google's Agent Development Kit landed

## The day ADK shipped

I remember the exact moment I opened the ADK docs. I had a half-finished side project that was a thousand lines of glue code wrapping a Gemini call, a tool router, a retry loop, and a small parser that tried to keep the model honest. After about an hour of reading, I deleted most of it.

The fastest way to build a Gemini agent that calls MCP tools is now two imports and a config object. That is not a marketing line. That is the actual diff in my editor. The agent loop, the tool dispatch, the model wiring to Vertex AI, all of it now lives inside the kit. I bring the prompt, the tools, and the rules. The kit brings the rest.

The Google I/O 2026 keynote framed ADK as the official path for Gemini agents. For me it was the moment that turned a slow side hustle into a sprint. I shipped seven agents in six weeks. Some are tiny. Some are real. All of them taught me something about what production Gemini code wants to look like.

## The 6-week sprint

Here is what landed, one line each, every one of them running on Vertex AI Gemini 2.5 Flash through ADK:

- **gemini-bright-agent**: research analyst on Bright Data MCP. Pulls SERP, scrapes pages, returns cited answers. Streamlit on Cloud Run.
- **gemini-rpa-agent**: triages failed RPA workflows. Given a run id, finds the breaking step, returns the verbatim error payload and a retry command.
- **gemini-data-agent**: MongoDB query helper over an MCP server. Natural language in, real query and result rows out.
- **gemini-splunk-agent (ops)**: SPL generator for ops dashboards. Reads index docs over MCP, drafts the search.
- **gemini-splunk-agent (security)**: same shape, security-focused. Looks at auth failures, anomalies, alert tuning.
- **gemini-splunk-agent (devx)**: developer-facing variant. Wraps common SPL snippets so engineers stop pinging the data team.
- **gemini-eval-agent**: auditor that talks to Arize Phoenix over MCP. Reads traces, flags weak agents, suggests prompt fixes.

There are a couple more in flight that I am not counting yet (a connector and a pipeline runner). What matters is the rhythm. Once the first one was real, the rest came fast because I stopped solving the same three problems over and over.

## The labeled-section output pattern

Every single one of these agents writes its final answer in named sections. The bright-data agent uses ANSWER, SOURCES, KEY QUOTES, CONFIDENCE, NEXT STEP. The RPA agent uses ANSWER, EVIDENCE, ROOT CAUSE, REMEDIATION, NEXT STEP. The Splunk variants use ANSWER, QUERY, EXPLANATION, RISKS. The shape changes per domain. The discipline does not.

Why bother? Because freeform LLM output is a downstream problem. The minute someone wants to render the answer in a UI, log it, eval it, or pipe one agent into another, that paragraph of helpful prose becomes a parsing nightmare. Labeled sections give me four wins on day one:

1. The model is forced to commit. If I ask for EVIDENCE and CONFIDENCE separately, it cannot smear them into one mushy sentence.
2. The UI is trivial. I split on the section header and render each block however I want. Markdown, cards, copyable code, whatever.
3. Evals get cheap. I can score CONFIDENCE calibration separately from ANSWER accuracy.
4. Agent-to-agent calls just work. The eval agent reads other agents' outputs by section name. No regex gymnastics.

I write the section list in the system prompt, give one tiny example, and Gemini 2.5 holds the format almost without exception. The 2.5 series got noticeably better at instruction following on long contexts, which is the only reason this pattern is practical at all.

## The MCP stub pattern

Every agent in the list ships with a local stub MCP server. The stub returns hand-written fixtures. One environment variable swap points the agent at a real account: a real Bright Data token, a real Splunk endpoint, a real Mongo URI, a real Phoenix collector.

This sounds like a small thing. It is not.

A stub means I can demo the agent on a plane. A stub means a reviewer on GitHub can clone the repo, run one command, and watch it work without signing up for a vendor account. A stub means my CI can run the full agent loop on every commit without burning real API quota. A stub means when the real upstream goes down, I can keep building.

The discipline is to make the stub return data with the same shape and the same field names as the real tool. The agent should not know the difference. If the real Splunk MCP returns `results` as a list of dicts with `_time` and `host`, the stub does the same. I have caught real bugs this way that would have hit me later in prod.

## The verbatim-quote rule

Of every rule I have written into these agents, this is the one I will fight for. Numbers, dates, error payloads, log lines, any quantitative or factual claim must be byte-for-byte from a tool result. No paraphrase. No rounding. No filling in plausible context.

The bright-data agent has it in the prompt as a hard rule: KEY QUOTES are exact excerpts from the scraped page with the source URL. The RPA agent has it as: EVIDENCE is byte-for-byte from the tool output. Same idea.

Why is this the rule I care about most? Because hallucinated numbers are the single fastest way to lose a user's trust forever. An agent that says "the workflow failed at step 4 with a timeout" when actually it was step 7 with a 502 from the upstream, is worse than no agent. A user who catches that lie once stops trusting every future output, even the right ones.

Verbatim quoting also forces the agent to actually use its tools. If the prompt demands an exact quote and the model has no tool result to draw from, it has to either go fetch one or admit it does not know. Both outcomes are better than confident bluffing.

In practice, I pair this rule with a tiny post-check: scan the agent's EVIDENCE block, confirm each quoted snippet appears in the raw tool output. If it does not, I flag the response and retry with a stricter instruction. It is the cheapest guardrail with the biggest payoff.

## What I want from Google I/O 2026 (next)

I am building on ADK every day, so I have a small wishlist:

- **Better cost tracking inside ADK.** Every agent run should report tokens in, tokens out, cache hits, and dollar estimate by default. I should not have to wire my own logger.
- **Native trace export to standard tooling.** OpenTelemetry GenAI conventions are stabilizing. ADK should emit them out of the box so I can ship traces straight to Arize, Langfuse, or my own collector without translation glue.
- **Sharper MCP tool errors.** When an MCP tool fails, I want a typed error surface that the model can reason about, not a stringified stack trace dumped into the conversation. Tell the agent what to do next.

If even one of these lands at the next I/O, the next six weeks will be wilder than the last six.

## Try it yourself

The two cleanest entry points if you want to see the pattern end to end:

- [gemini-bright-agent](https://github.com/MukundaKatta/gemini-bright-agent): research analyst on Bright Data MCP, full labeled-section output, stub mode included.
- [gemini-rpa-agent](https://github.com/MukundaKatta/gemini-rpa-agent): RPA workflow triage with byte-for-byte EVIDENCE, four MCP tools, stub mode included.

Both have a Streamlit front end so you can poke at them in a browser. Clone, install, point at the stub, ask a question. If you want to see the real version, drop in your own credential in the one env var the README calls out.

If you build something with this shape and want a second pair of eyes, my GitHub is open. I would love to see what the next person does with these patterns.

---

Mukunda Katta. Shipping small Gemini agents in public.
