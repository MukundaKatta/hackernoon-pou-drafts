# cachebench: Find Out You Lost Your Prompt Cache Before the Bill Does

Prompt caching can cut your LLM cost by around 70% if you do it right. Most apps do not.

They miss because the prefix shifts by one whitespace token. Or a deploy adds a timestamp to the system prompt. Or someone reorders two tool definitions. The provider quietly stops serving from cache. The numbers in your logs look the same. The latency looks the same. The cost does not.

You find out at month end.

I built cachebench so you find out the same hour.

## What it is

cachebench is a Python library that wraps your LLM calls and tracks cache behavior per request. Hit ratio. Read tokens. Creation tokens. Estimated dollars saved. A stable hash for the prefix so you can tell which system prompt regressed.

It works with Anthropic, OpenAI, and AWS Bedrock today.

Install it:

```
pip install cachebench
```

The whole API is built around one object, `CacheTracker`, and one method, `wrap`. You hand it your provider's create function, you get back the same function with a meter strapped to it.

```python
from anthropic import Anthropic
from cachebench import CacheTracker, Provider

client = Anthropic()
tracker = CacheTracker(provider=Provider.ANTHROPIC, miss_alert_threshold=0.6)
create = tracker.wrap(client.messages.create)
```

That is it. You call `create` exactly like you would call `client.messages.create`. The wrapped version forwards arguments, returns the same response object, and pulls the cache fields out of the usage block on the way through.

## How it works

The interesting part is what you do with the tracker after a few calls.

```python
response = create(
    model="claude-sonnet-4-20250514",
    max_tokens=200,
    system=[{"type": "text", "text": LONG_PROMPT, "cache_control": {"type": "ephemeral"}}],
    messages=[{"role": "user", "content": "Hello"}],
)

print(tracker.aggregate())
# {'calls': 1, 'hit_ratio': 0.94, 'cost_saved_usd': 0.012, ...}

print(tracker.by_prefix())
# {'<hash-of-system-prompt>': {'calls': 1, 'hit_ratio': 0.94, ...}}
```

`aggregate()` rolls everything up. `by_prefix()` groups by a stable hash of the cached prefix, so if you have three different system prompts in production, you see three rows. When one of them drops to a 10% hit ratio after a deploy, that row is the smoking gun.

There is also a miss-aware retry policy. Providers have eventual-consistency windows on cache writes. A request right after a fresh write can miss even when the prefix is correct. `CachePolicy.miss_aware()` retries once after a short delay so you do not pay full price on the second token a user types.

Alerts run through a threshold. Set `miss_alert_threshold=0.6` and any wrapped call that falls below that hit ratio fires a handler. Default handler writes to stderr. You can pass your own callable for Slack or PagerDuty.

Async paths work the same way. The wrapper detects coroutines and wraps them without changing the signature.

## Why it matters

Every provider reports cache stats in their own shape. Anthropic returns `cache_read_input_tokens` and `cache_creation_input_tokens` on the usage object. OpenAI tucks it inside `prompt_tokens_details.cached_tokens`. Bedrock follows the Anthropic shape but only on the `AnthropicBedrock` client. If you want a single dashboard across all three, you write a normalizer. I wrote it once so you do not have to.

The other thing providers do not give you is a stable identity for a prefix. They tell you how many tokens were read from cache on this call. They do not tell you "this is the same system prompt you sent 4000 times yesterday and it used to hit 94%". cachebench hashes the cached segments and gives you that grouping for free.

This is the loop I wanted: deploy a prompt change, run a few hundred real requests, check `by_prefix()`, see if the new prefix is hitting cache as well as the old one. If it is not, roll back before the next billing window.

## What is missing

This is v0.1.0. The Proof of Usefulness report scored it 38, which was the lowest of the nine libraries I submitted. I think that is fair. Some honest gaps:

- The cost math uses static pricing tables. If a provider changes prices, the dollar numbers drift until I update the table. You can pass `pricing={...}` to override.
- The OpenAI integration only reads cache fields that the API actually populates today. If they extend the schema, I have to catch up.
- There is no UI. No dashboard. `aggregate()` and `by_prefix()` return dicts. You wire them into whatever you already use for metrics.
- It is observational. It does not cache responses on your behalf, does not proxy, does not rewrite prompts. If you want a router, this is not it.
- I have not stress-tested the async path under heavy concurrency. It works in my tests. I would not bet a Black Friday on it yet.

If those things matter to you, hold off. If you just want to know whether your prompt cache is actually hitting in production, the library is small enough that you can read it in one sitting and decide.

## Try it

```
pip install cachebench
```

Source: https://github.com/MukundaKatta/cachebench

PyPI: https://pypi.org/project/cachebench/

License: MIT.

Issues and PRs welcome. The hardest part of this library is keeping up with how providers report cache stats, so if you spot a field I am missing, a one-line issue is the most useful thing you can send me.

Tags: #ai #prompt-caching #llm #python #open-source #proof-of-usefulness #cost-optimization #developer-tools
