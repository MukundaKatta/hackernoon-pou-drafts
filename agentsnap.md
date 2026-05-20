# I Got Tired of Silent Agent Regressions, So I Built Snapshot Tests for Tool Calls

The bug that pushed me over the edge looked like this. An agent that used to call `search` then `summarize` started calling `summarize` then `search` after a small prompt tweak. Output looked fine in spot checks. Two days later a customer noticed it was summarizing stale cache instead of fresh results. No test caught it, because no test could see the call order. That is why I built **agentsnap**.

## What it is

agentsnap is a tiny library that does for AI agent runs what Jest snapshots do for React components. You wrap your tool functions, run the agent inside a recorder, and save the resulting trace to disk. The next time the test runs, the new trace gets diffed against the saved one. If the tool order changes, or a tool appears or disappears, the test fails.

Install it as a dev dependency:

```bash
npm install --save-dev @mukundakatta/agentsnap
```

The whole API is five functions. You will mostly use three of them: `traceTool`, `record`, and `expectSnapshot`.

## How it works

Here is a real test, close to what is in the repo.

```js
import { record, traceTool, expectSnapshot } from '@mukundakatta/agentsnap';

const search = traceTool('search', async ({ q }) => fetchResults(q));
const summarize = traceTool('summarize', async ({ docs }) => llmSummarize(docs));

async function agent(question) {
  const docs = await search({ q: question });
  return summarize({ docs });
}

test('research agent stays on rails', async () => {
  const trace = await record(() => agent('What is RLHF?'));
  await expectSnapshot(trace, '__snapshots__/research.snap.json');
});
```

First run, it writes the snapshot file. Every run after that, it loads the baseline and diffs the live trace against it. The diff has five outcomes: `PASSED`, `OUTPUT_DRIFT` (the args or results shifted a bit, soft warning), `TOOLS_REORDERED`, `TOOLS_CHANGED`, and `REGRESSION`. The last three fail the test.

A couple of details that I cared about while building it. Outside of `record()`, the wrapped tools pass through with zero overhead, so you can leave the wrappers on in production. The diff is plain JSON, so you can review it in a pull request the same way you review any other snapshot file. And there is a `formatDiff` helper that renders the diff in color in your terminal, which makes the failure output actually readable.

For more involved cases you can drop down to `diff(baseline, current)` and write your own pass or fail logic. I have used this to ignore a specific argument key that drifts on purpose (timestamps).

## Why this matters now

The judges at Proof of Usefulness gave agentsnap a score of 61.33 and put it in the "You're In Business" tier. The bit of the report I keep thinking about is this line:

> agentsnap targets a highly relevant and emerging pain point in AI engineering: silent regressions in agent tool-calling sequences. The proposed solution of adapting Jest-like snapshot testing to LLM agent traces is elegant, highly practical, and perfectly timed for current market needs.

I think the timing part is the real story. Right now a lot of teams are shipping agents to production with maybe an eval set of question-answer pairs, and nothing that watches the trace. When the model provider pushes a silent update, or someone edits the system prompt, the answers can still look correct while the tool-call order shifts underneath. You only find out when something breaks downstream.

The reason snapshot testing works for this is the mental model is already there. Every JavaScript engineer who has touched a Jest test knows what `toMatchSnapshot` does. Telling them "same idea, but the snapshot is an agent trace instead of a rendered component" takes about ten seconds. They do not need a new framework or a new dashboard. They just need the test to fail in CI when the trace changes.

## What is still missing

The library is at v0.1.2. There are 37 unit tests and the core API is stable, but I want to be honest about the rough edges.

Right now you have to wrap your tools yourself with `traceTool`. If you use the Anthropic SDK or the OpenAI SDK or an MCP client directly, you do a small amount of glue work to get tool calls into the recorder. The v0.2 plan is to ship adapters for Anthropic, OpenAI, and MCP so the wrapping disappears. There is also no built-in support for redacting secrets from the trace before it gets written, which matters if your tool arguments include API keys. For now I handle that with a custom `diff` step, but it should be a first class option.

The other thing I want to add is a way to mark certain fields as "expected to drift" (LLM-generated text in tool arguments is the obvious case), without losing the strict checking on structure and order. Today you either get an exact match or you write your own diff.

## Try it

If you build agents and you do not have a regression test for the trace, this is roughly an afternoon of work to add.

```bash
npm install --save-dev @mukundakatta/agentsnap
```

Source is on GitHub at [MukundaKatta/agentsnap](https://github.com/MukundaKatta/agentsnap). License is MIT. Issues and pull requests are open. If you try it and the API gets in your way, tell me. I would rather find out from you than from a production trace that nobody recorded.

---

Tags: #ai-agents #testing #open-source #proof-of-usefulness #typescript #llm #developer-tools #mcp
