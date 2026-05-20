# I asked the LLM for JSON. It gave me JSON wrapped in triple-backticks with an extra comma.

That is the moment I built agentcast.

The prompt was fine. The schema was fine. The model was a good one. And I still got back:

````
Sure! Here is the product you asked for:

```json
{
  "name": "Coffee Mug",
  "price": "twelve",
  "in_stock": true,
  "tags": ["kitchen", "ceramic",],
}
```
````

Three things wrong in one response. Prose wrapper. `"twelve"` instead of `12`. Trailing comma. My code just did `JSON.parse(response)` and threw. I added a regex to strip the fences. Then a `try/catch`. Then a retry. Then a "tell the model what went wrong" branch. Two days later I had 80 lines of ad-hoc retry logic in a file called `parseJsonProbably.ts`, and I had it in three different repos.

So I pulled the pattern out into a tiny library.

## What agentcast is

It is a validate-and-retry loop for LLM JSON. One function. Bring your own LLM. Bring your own validator. Zero runtime dependencies.

```bash
npm install @mukundakatta/agentcast
```

```js
import { cast, adapters } from '@mukundakatta/agentcast';
import { z } from 'zod';

const productSchema = z.object({
  name: z.string(),
  price: z.number(),
  in_stock: z.boolean(),
  tags: z.array(z.string()),
});

const product = await cast({
  llm: async (messages) => callYourModel(messages),
  validate: adapters.zod(productSchema),
  prompt: 'Generate one example product.',
  maxRetries: 3,
});
```

If `cast` returns, `product` has the shape you asked for. If it cannot get there in 3 tries, it throws `CastError` with the full attempt history so you can see what the model actually said.

That is the whole pitch.

## How it works

There are three steps inside `cast`. None of them are clever. That is on purpose.

**Step 1: extract.** The model rarely returns clean JSON. It returns JSON inside a sentence. JSON inside a ```` ```json ```` fence. JSON inside a plain ```` ``` ```` fence. Or just the JSON object with text before and after. `extractJson` tries each strategy in order: parse the whole text, then look for a fenced block, then find the largest balanced `{...}` or `[...]` substring in the prose.

**Step 2: validate.** Whatever came out of step 1, run it through your validator. The adapters are thin:

```js
// any safeParse-style validator (zod, valibot)
adapters.zod(z.object({ name: z.string() }))

// a predicate function with a custom error builder
adapters.fn(
  (v) => typeof v?.score === 'number' && v.score >= 0,
  (v) => `score must be >= 0, got ${v?.score}`
)

// a built-in shape check if you want zero deps end-to-end
adapters.shape({ name: 'string', age: 'number', tags: 'array' })
```

If the validator returns `{ valid: true, value }`, you get `value` back and you are done.

**Step 3: retry with the error as feedback.** This is the part most people skip. If validation fails, agentcast appends the model's bad response to the message history, then appends a new user message telling it what was wrong. So attempt 2 sees:

- your original prompt
- the model's prose-wrapped, missing-field response from attempt 1
- a message that says "the response failed validation: `tags` is required"

Models are much better at fixing their own mistakes than at getting it right the first time. The feedback loop is what makes this work.

After `maxRetries` failed attempts, `cast` throws `CastError` with `attempts`, `lastError`, `lastText`, and `lastParsed` so you can debug instead of guess.

## Why this matters

Structured output is still the number one broken thing in agent stacks. People ship agents to production and then add Sentry alerts for `JSON.parse` failures.

Providers have noticed. OpenAI has JSON mode. Anthropic has tool-use as a structured-output trick. Vercel's `ai` package has `generateObject`. They all help. They all still fail sometimes, and when they fail they do not repair. They just hand you back a string and a stack trace.

agentcast sits in front of any LLM call. It does not care which provider, which SDK, which model. If you can write `async (messages) => string`, it works. That is the whole interface.

The Proof of Usefulness report scored it 53.76. The note I cared about was "highly relevant and ubiquitous developer pain point." That matches what I felt writing the same retry loop for the third time.

## What is missing

I want to be honest about v0.1 state. The repair pass is intentionally small and there are real cases it does not cover yet:

- **No streaming.** If the model is streaming JSON, you have to wait for the full string before `cast` can extract. A streaming variant is on the v0.2 list.
- **No JSON-mode hint generation.** If you are on OpenAI or Anthropic and want to set the provider's JSON-mode params, you have to set them yourself. I want to add an opt-in flag that builds those for you.
- **No cost cap.** Right now `maxRetries: 5` means up to 5 calls, full stop. It does not know the next call would blow a token budget. Cost-aware retry is on the list.
- **Repair is extract-only.** I do not try to fix the JSON before parsing. So `{name: "alice"}` (unquoted key) or `{"price": "twelve"}` (wrong type) is not repaired locally. It is sent back to the model with a feedback message. That is intentional; local repair lies about the model's behavior. But it is a tradeoff and some people want the lie.
- **No tool-call validation.** This is for the model's output shape, not the arguments it passes to tools mid-conversation. For that I wrote a sibling library called agentvet.

If you hit a case the extractor misses, file an issue with the exact LLM string and I will add a test.

## Try it

```bash
npm install @mukundakatta/agentcast
```

GitHub: https://github.com/MukundaKatta/agentcast
npm: https://www.npmjs.com/package/@mukundakatta/agentcast
PoU report: https://proofofusefulness.com/reports/agentcast-structured-output-for-any-llm-call

There is a `examples/demo-retry.js` you can run with `node` to watch the retry loop in action against a stubbed model that returns bad JSON twice and clean JSON the third time. It is the fastest way to see the message history grow with feedback.

License: MIT. Issues and PRs welcome. I read all of them.

If you have ever written `parseJsonProbably.ts`, this is the version you do not have to maintain.

#ai-agents #llm #structured-output #open-source #proof-of-usefulness #typescript #json-schema #developer-tools
