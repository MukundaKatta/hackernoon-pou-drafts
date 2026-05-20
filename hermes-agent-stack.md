---
title: "Wrapping Hermes Agent with agent-stack: six tiny libs for the boring parts"
published: false
description: "Hermes runs the agent loop. agent-stack handles the budgeting, egress, snapshots, validation, repair, and caps around it. Built for the Hermes Agent Challenge."
tags: devchallenge, hermesagentchallenge, opensource, typescript
---

# Wrapping Hermes Agent with agent-stack: six tiny libs for the boring parts

## The day my agent silently broke

A customer pinged me on a Tuesday. The reply from my agent looked fine. The tool call underneath did not. One field flipped from `user_id` to `userId` after a model swap, and my tool call started returning empty objects instead of throwing. The agent kept going. The user kept asking why their lookup felt off. I found out three days later from a support ticket.

That bug was not in the agent loop. The loop was perfect. The bug was around it. Argument shape changed. No one caught it because there was no snapshot to compare against. No validator to fail fast. No trace to diff.

That story is why I built agent-stack. Six small libs, one repo, one install. Each one fixes one thing that goes wrong outside the agent loop.

This post is my entry for the Hermes Agent Challenge. The pitch fits the contest brief: ride on top of an open source agent (Hermes from Nous Research), do not rebuild what already works, ship the boring guardrails as composable pieces.

## What agent-stack is

Six single-concern libraries. Each one shipped to npm and PyPI. Each one also ships as an MCP server. Zero runtime deps.

| Library | What it does |
| --- | --- |
| AgentFit | Token-aware context-window fitting |
| AgentGuard | Network egress allowlist for tool calls |
| AgentSnap | Snapshot tests for tool-call traces |
| AgentVet | Tool-arg validation with retry hints |
| AgentCast | Structured-output validate-and-retry for LLM JSON |
| AgentBudget | Per-run token and dollar caps |

Repo: https://github.com/MukundaKatta/agent-stack

Install all six in one line:

```bash
npm i @mukundakatta/agentfit @mukundakatta/agentguard @mukundakatta/agentsnap \
      @mukundakatta/agentvet @mukundakatta/agentcast @mukundakatta/agentbudget
```

Each lib does one thing. You can adopt one, you can adopt all six. They do not depend on each other. They do not depend on a specific LLM SDK. They wrap whatever agent loop you already have.

## Hermes is the agent. agent-stack is the safety net around it

Nous Research shipped Hermes as an open agent. Tool calls, planning, the loop, the prompt templates. All of that is solved. What is not solved in any agent codebase I have seen is the operational layer around it. The stuff that catches regressions, stops a runaway loop, blocks an exfiltration, repairs a malformed JSON response.

The Hermes Agent Challenge brief is direct about this: borrowing and improving on previous work is encouraged. So I am not rebuilding Hermes. I am wrapping it.

Here is the shape of the integration. Pretend `runHermesAgent` is your call into the Hermes loop (whatever shape it takes for you, this is the wrapper pattern):

```ts
import { snapshot } from "@mukundakatta/agentsnap";
import { guard } from "@mukundakatta/agentguard";
import { budget } from "@mukundakatta/agentbudget";
import { runHermesAgent } from "./hermes";

// Egress allowlist. Anything the agent's tools try to fetch
// outside this list throws.
const allowlist = guard({
  domains: ["api.openweather.com", "api.github.com"],
});

// Per-run cost cap. Throws BudgetExceeded if the run goes over.
const cap = budget({
  maxTokens: 50_000,
  maxUsd: 0.50,
});

// Snapshot wrap. First run records, later runs diff.
export const safeRun = snapshot("weather-agent", async (input) => {
  return await allowlist.run(() =>
    cap.run(() =>
      runHermesAgent(input)
    )
  );
});

// In your test
test("weather agent matches snapshot", async () => {
  await safeRun({ city: "Dallas" });
  // AGENTSNAP_UPDATE=1 to refresh the recorded trace
});
```

Three things just happened around the Hermes call.

First, `guard` wrapped every fetch the agent tools make. If the model hallucinates a URL or a compromised tool tries to POST somewhere weird, it throws before the bytes leave the box.

Second, `budget` tracks token and dollar spend across the whole run. If a planner loop goes wide, you get a clean exception instead of a $40 bill.

Third, `snapshot` records the full tool-call trace on the first run. Every later run diffs against it. The `userId` vs `user_id` bug from my Tuesday story would have failed the snapshot the moment the field renamed.

Add `cast` if you want LLM JSON output repaired and retried. Add `vet` if you want tool args validated with retry hints sent back to the model. Add `fit` if you have a long-conversation agent and need token-aware truncation. All of them stack the same way.

## Why this riffs on Hermes

The Hermes design philosophy from Nous Research is that the agent should stay open, hackable, and composable. You wire it into your system. You do not get locked into a platform.

That only works if the operational layer is also composable. If I want to add a snapshot test, I should not need to fork Hermes. If I want to cap dollar spend, I should not need a vendor SDK. If I want an egress allowlist, I should not need a service mesh.

agent-stack is the cheapest possible version of those six concerns. Each lib is small enough to read in one sitting. Each one runs in the same process as the agent. There is no daemon, no sidecar, no SaaS.

That matches Hermes. Open loop in the middle. Open guardrails around it. You own the whole stack.

## What is still missing

This is v0.1.x. Honest list of gaps:

- AgentSnap diffs are JSON-level today. No semantic diff for free text yet.
- AgentBudget pricing tables cover Anthropic, OpenAI, Bedrock. Hermes-specific pricing is a manual override right now.
- AgentGuard does not yet intercept raw `node-fetch` if you import it before the guard wraps. Use the wrapper pattern shown above.
- The MCP server variants are thin. They expose each lib as a tool. Battery-included examples are still on the punch list.

If any of these block you, the libs are MIT and the issue tracker is open. I would rather merge your patch than ship a v0.2 alone.

## Try it

GitHub: https://github.com/MukundaKatta/agent-stack

npm packages under `@mukundakatta/agent*`. PyPI packages under the same names. License is MIT across the board. Paper is up on Zenodo with a DOI if you want to cite it: https://doi.org/10.5281/zenodo.20074702

Live demo runs on a Hugging Face Space: https://huggingface.co/spaces/mukunda1729/agent-stack-demo

The whole point of this submission is that the work is already open. Hermes is open. agent-stack is open. The six libs are small enough that you can read them in an afternoon and rip out the parts you do not want.

If you build something on top of Hermes for the challenge, try wrapping it with one of these. Start with AgentSnap. The first time a model swap silently changes a tool call, you will be glad the test failed instead of a customer.

Built for the Hermes Agent Challenge. Submission by Mukunda Katta.
