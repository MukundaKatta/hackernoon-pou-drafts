# agenttrace: Cost and Latency Tracking for Agent Runs in One Function

You ran the agent ten times today. Each run made eight LLM calls. A few hit Claude Sonnet, a few hit GPT-4.1, one or two went through a local Groq model. Then finance walks over and asks: "what did this cost us?"

Now you have a choice. Open a spreadsheet, copy token counts out of logs, look up pricing tables, pray you got the model names right. Or wire up agenttrace and get a structured report out of every run.

I built agenttrace because I kept doing the spreadsheet thing and I was tired of it.

## What it is

agenttrace is a small Node library that wraps your agent runs and tracks two things: cost and latency. Per step, per run. No SaaS account. No big dashboard. No agent framework lock-in. You drop it next to your existing code.

Install:

```bash
npm install github:MukundaKatta/agenttrace
```

It is not on npm yet. Install from GitHub until v0.1.0 ships.

The whole API is built around `withRun`. You open a run context, do your agent work inside it, and read back a structured `Run` object when you are done. Any `measure` or `measureLLM` calls inside the block attach to that run through AsyncLocalStorage. No manual context threading.

## How it works

The smallest useful example:

```javascript
import { withRun, measureLLM } from "@mukundakatta/agenttrace";

const { run, value } = await withRun({ name: "summarize-doc" }, async () => {
  const reply = await measureLLM(
    "summarize",
    "claude-sonnet-4",
    async () => anthropic.messages.create({
      model: "claude-sonnet-4-5",
      messages: [{ role: "user", content: "..." }],
    }),
  );
  return reply.content[0].text;
});

console.log(run.summary());
```

That prints a one-line report: the run name, total steps, total latency, total cost, and a row per step. If you want the structured version for logs or a dashboard, call `run.toJSON()` and ship it wherever you ship JSON.

Mixing LLM calls and tool calls works the same way:

```javascript
const { run } = await withRun({ name: "research" }, async () => {
  const docs = await measure("retrieve", () => ragStore.fetch("..."));
  const summary = await measureLLM("summarize", "gpt-4.1-mini",
    () => openai.chat.completions.create({ /* ... */ }));
  await measure("write", () => fs.writeFile("out.md", summary));
});
```

Tool calls get latency tracking. LLM calls get latency plus token usage plus dollar cost. The `Run` object carries `totalCostUsd`, `latencyMs`, `totalUsage` (input, output, cache write, cache read), and a list of steps. Each step carries its own copy of the same fields.

Pricing is built in for Anthropic Claude families, OpenAI GPT-4.1 / GPT-4o / o-series, Gemini 2.5 and 2.0, Grok 4, and free-tier providers like Groq, Cerebras, Ollama, and OpenRouter free models. Model name matching uses longest-prefix lookup, so `claude-sonnet-4-5-20260101` finds the `claude-sonnet-4` row.

If you run a private model or your contract pricing differs, register your own rates:

```javascript
import { setPricing, costOf } from "@mukundakatta/agenttrace";

setPricing("my-private-llm", { input: 1.5, output: 6 });
costOf("my-private-llm", { input: 10_000, output: 5_000 });
```

Prices are USD per million tokens. The result is a number. You decide where to put it.

## Why it matters

Most observability tools for agents want a big integration. You install an SDK, sign up for a hosted backend, instrument every layer, accept that your traces live in someone else's database. Phoenix, LangSmith, Langfuse, all of them want that footprint. They are great tools when you need that footprint. Most days I do not.

What I want is one function I can wrap around an agent run, so I can answer two questions: how long did it take, and what did it cost. That is it.

agenttrace is that one function. It composes. The `Run` JSON drops straight into Pino, Winston, OpenTelemetry, a flat file, a Postgres row. You can pipe `totalCostUsd` into a Slack alert. You can grep p95 latency out of yesterday's logs. You keep your stack. You add visibility.

It also sits next to the other small libs I built for the same reason: agentfit for context window truncation, agentguard for egress allowlists, agentvet for tool arg validation, agentsnap for snapshot tests, agentcast for structured output. None of them ask you to rebuild your agent. They each do one thing and stay out of the way.

## What is missing

v0.1.0 means v0.1.0. Things to know before you ship it in production:

- Not on npm yet. Install from GitHub.
- Node 20 plus. ESM only.
- 26 tests pass. Coverage is not 100 percent.
- p50 and p95 across runs require you to feed `run.toJSON()` into your own aggregation. The library reports per-run latency. Multi-run percentiles are your call on where to store and how to roll up.
- Built-in pricing reflects public rate cards. Enterprise contracts and discounts need `setPricing`.
- Streaming responses need a custom `extractUsage` if your SDK does not expose token counts the standard way.

If any of those are blockers, file an issue or send a PR. The surface area is small enough that fixes land fast.

## Try it

GitHub: https://github.com/MukundaKatta/agenttrace

Install:

```bash
npm install github:MukundaKatta/agenttrace
```

License: MIT.

It is a Proof of Usefulness Hackathon entry. The PoU report is here: https://proofofusefulness.com/reports/agenttrace-cost-and-latency-tracking-for-agent-runs. Score 45.

If you run agents and you do not know what they cost, wrap one with `withRun` and see what falls out. Worst case you delete the import. Best case you stop opening that spreadsheet.

---

Tags: #ai-agents #observability #cost-tracking #open-source #proof-of-usefulness #llm #developer-tools
