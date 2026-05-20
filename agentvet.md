---
title: "agentvet: Stop LLMs From Calling Your Tools With Garbage Args"
tags: [ai-agents, tool-use, validation, open-source, proof-of-usefulness, typescript, llm, developer-tools]
---

# agentvet: Stop LLMs From Calling Your Tools With Garbage Args

The agent picks a tool. It picks the right tool. Then it fills the args wrong.

Maybe a string where a number was expected. Maybe a missing required field. Maybe a date in the wrong format. The function throws somewhere deep in the stack. The trace dies. There is no retry. The agent moves on, confused, sometimes looping, sometimes giving up.

Or worse. The args are wrong in a way that the function does not catch. It partially succeeds. It writes a row to a table with a null where a foreign key should be. It sends a half-formed email. It charges a card with the wrong amount. Now state is corrupted and the agent has no idea.

I kept hitting this same wall. Every time I shipped a tool to an agent, I had to write a validation prelude inside the tool itself. And when validation failed, I had to think about what to tell the model so it could try again. Most of the time I did not bother. The tool would just throw a TypeError and the run would die.

I wrote agentvet to fix that.

## What it is

agentvet wraps a tool function with a schema. If the LLM calls the tool with bad args, it throws a `ToolArgError` before the function runs. The error carries a structured retry hint that you can hand straight back to the model on the next turn.

```bash
npm install @mukundakatta/agentvet
```

The whole surface is one function. You take your tool, you wrap it in `vet()`, you give it a name and a schema. That is the whole library.

## How it works

Here is a tool that sends an email. I do not want the agent to call it with an empty `to` field. I do not want it called with a non-string subject. I want the model to fix its own mistake instead of me dropping the trace.

```typescript
import { vet, ToolArgError } from "@mukundakatta/agentvet";
import { z } from "zod";

const sendEmail = vet({
  name: "send_email",
  schema: z.object({
    to: z.string().email(),
    subject: z.string().min(1),
    body: z.string().min(1),
  }),
  fn: async (args) => {
    return await mailer.send(args);
  },
});
```

Now I call it from my agent loop. If the model gets the args right, it runs. If the model gets the args wrong, it throws a `ToolArgError`. I catch that error and feed `err.toLLMFeedback()` back into the next turn.

```typescript
try {
  const result = await sendEmail(toolCall.args);
  messages.push({ role: "tool", content: result });
} catch (err) {
  if (err instanceof ToolArgError) {
    messages.push({
      role: "tool",
      content: err.toLLMFeedback(),
    });
  } else {
    throw err;
  }
}
```

The feedback string is shaped for the model. It names the tool, it lists which fields failed, and it says what was expected. The model reads it and tries again with corrected args. No state was touched. No half-write happened. The mailer was never called.

agentvet ships with adapters for Zod, Valibot, custom predicates, and a built-in shape checker if you do not want a schema dependency at all. There is also a `validate()` helper for one-off inline checks, and a small CLI for linting tool definitions before you ship them.

## Why it matters

Every agent framework I have read still treats tool calling as "trust the LLM." The framework hands the model a tool schema, the model emits arguments, the framework calls the function. If the args are wrong, you are on your own.

That is the gap where production agents break. Not in the planning, not in the reasoning, but in the boring part where the model gets a field name slightly wrong and the whole run falls over.

The validation gap is small to describe and huge in practice. You either pay the cost up front by writing schemas, or you pay it later in customer support tickets and corrupted rows. agentvet picks the up-front option and makes it cheap.

The other thing I care about: the retry hint. A lot of tool wrappers will validate args and throw a useful error to you, the developer. agentvet throws an error to the model. The string is written for the LLM to read. That is what closes the loop. Without the structured hint, you still have to write the retry prompt yourself, and most people skip it.

## What is missing

v0.1.0 is small on purpose. The wrap function works, the adapters work, the error shape is stable, the CLI lints definitions. I use it in my own agents.

What is not in v0.1.0:

- No built-in retry runner. You still write the loop. I want to keep the library unopinionated about how you call your model.
- No telemetry or tracing hooks. I have a sibling library, agentsnap, that does that, and I want to keep these separate.
- No automatic schema generation from TypeScript types. You write the schema. I might add this later if people want it.
- Python port is not done yet. Same author, same patterns, but I have not shipped it.

If you find something that should be in the core, open an issue. I read them.

## Try it

- GitHub: https://github.com/MukundaKatta/agentvet
- npm: https://www.npmjs.com/package/@mukundakatta/agentvet
- Proof of Usefulness report: https://proofofusefulness.com/reports/agentvet-validate-tool-args-before-execution (score 49.23)

MIT licensed. No telemetry. No phone home. Wrap your tools, send the feedback string back to the model, stop dropping runs because of a bad arg.

If you ship agents and you have ever stared at a stack trace wondering why your tool got called with `user_id: undefined`, this is the library I wish I had a year ago.

#ai-agents #tool-use #validation #open-source #proof-of-usefulness #typescript #llm #developer-tools
