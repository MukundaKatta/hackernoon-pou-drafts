# bedrock-kit: the AWS Bedrock wrapper I kept rewriting in every project

Tags: #ai #aws-bedrock #python #open-source #proof-of-usefulness #llm #developer-tools

Every time I start a Bedrock project, I write the same three helpers. A retry loop because the throttle exceptions show up the minute traffic looks real. A cost calculator because boto3 hands you token counts but no dollars. A JSON cleaner because models keep wrapping objects in markdown fences or trailing a comma I don't want to deal with.

By the third project I had three janky helpers in three repos, all slightly different, all written at 11pm. So I pulled them out into a library and called it bedrock-kit.

## What it is

bedrock-kit is a small Python wrapper around the boto3 Bedrock client. It does three things:

1. Adaptive throttle handling with backoff and jitter.
2. Per-call cost tracking that knows about cache hits and misses.
3. Structured output parsing with JSON repair.

That's it. No agent loops. No prompt manager. No proxy server. No router. It is a client wrapper that turns the four lines of glue you write every time into one constructor.

Install it:

```bash
pip install "bedrock-kit[boto]"
```

The shape of the API:

```python
from bedrock_kit import BedrockClient, AdaptiveThrottle, CostLedger

ledger = CostLedger()
client = BedrockClient(
    region="us-east-1",
    retry=AdaptiveThrottle(max_attempts=8, base_delay=1.0, max_delay=30.0),
    cost_ledger=ledger,
)
```

The constructor takes a region, a retry policy, and a cost ledger. The retry policy is its own object so you can swap it out or pass `None` if you want raw behavior. Same with the ledger. Both are optional. The defaults are the ones I would have set anyway.

## How it works

Here is a full call. Make the request, get the text back, get the cost back, and if you asked for structured output, get a Pydantic model back already parsed.

```python
from pydantic import BaseModel
from bedrock_kit import JsonSchema

class Sentiment(BaseModel):
    label: str
    confidence: float

resp = client.invoke(
    model_id="anthropic.claude-sonnet-4-5",
    messages=[{"role": "user", "content": "Classify: 'this is great!'"}],
    response_schema=JsonSchema(Sentiment, max_repair_attempts=2),
)

print(resp.text)                  # raw text
print(resp.usage.input_tokens)    # tokens in
print(resp.cost_usd)              # dollars for this call
print(resp.parsed)                # Sentiment(label=..., confidence=...)
print(ledger.total_usd)           # running total across all calls
```

Three things happen behind that one call.

First, if Bedrock returns `ThrottlingException` or `TooManyRequestsException`, the client backs off and retries with jitter up to the cap you set. Permanent errors like a bad model id are not retried. Boto3 has its own retry config but it does not know which Bedrock errors are worth retrying versus which mean stop.

Second, when the response comes back, the cost is computed from the token counts and the pricing table baked into the library. If the response includes cache read or cache write tokens, those get priced separately at the cache rate. The total is stamped on the response and added to the ledger.

Third, because I passed a `response_schema`, the client pulls JSON out of the model output, strips fences and stray prose, parses it, and validates against the Pydantic model. If the first parse fails, it tries a local repair pass. If that fails, it can re-ask the model with the error message attached, up to the repair limit I set.

If you want to test code that uses bedrock-kit without paying for any calls, you pass a fake client in:

```python
class FakeClient:
    def converse(self, **kwargs):
        return {
            "output": {"message": {"content": [{"text": "stub"}]}},
            "stopReason": "end_turn",
            "usage": {"inputTokens": 1, "outputTokens": 1},
        }

client = BedrockClient(client=FakeClient())
```

Everything downstream works the same way. Cost, retry, parsing.

## Why it matters

Bedrock pricing is not one table. It is one table per region. And cache reads are cheaper than cache writes which are cheaper than fresh input tokens. If you are paying any attention to spend, you need that math correct on every call, not at the end of the month when the bill shows up.

Boto3 does not help with this. It returns token counts and a model id. The pricing lives in AWS docs and a console page you cannot script against without effort. So most projects either skip cost tracking entirely or hard-code a number that drifts the next time pricing changes.

bedrock-kit ships the pricing table inside the package, keyed by model id and region, with separate entries for cache read and cache write tokens. When AWS updates pricing, the library updates. You do not maintain that file. You also do not write a third janky helper.

The throttle piece is the same shape of problem. AWS gives you a generic boto3 retry config that does not know which Bedrock-specific errors are transient. So you either retry too aggressively and burn quota or not enough and drop requests. The `AdaptiveThrottle` class encodes the right list, with backoff and jitter, and lets you tune the caps.

And the JSON repair piece is the smallest of the three but the most annoying. Models put their JSON inside ```json fences, or add a sentence before it, or write a trailing comma. A local cleanup pass catches most of it. For the rest, you re-ask the model with the validation error. bedrock-kit does both in order.

## What's missing

This is v0.1. I want to be honest about that.

No streaming. No cancellation. No OpenTelemetry hooks (planned for v0.2). No support for image generation models. No multi-provider fallback. No prompt template engine. The public API may change before v1.0.

The pricing table covers Anthropic models on Bedrock with cache awareness. Other vendor families on Bedrock are not priced yet. If you call them, the response still comes back with text and tokens, but `cost_usd` will be `None`.

I am keeping the scope tight on purpose. There is a sibling library called driftvane that handles drift detection, and a couple of other small libraries I have been splitting out instead of bolting onto this one. Small libraries are easier to read, easier to test, and easier to drop.

## Try it

GitHub: https://github.com/MukundaKatta/bedrock-kit

PyPI: `pip install bedrock-kit`

License: Apache-2.0.

If you have ever copied your own Bedrock retry loop from one repo into the next, this is the library I wish you had had. And the one I wrote so I would stop copying mine.
