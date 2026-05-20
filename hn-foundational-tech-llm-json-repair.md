# llm-json-repair: A Tiny Rust Crate That Saves Your Second LLM Call

Tags: #ai #rust #llm #json #open-source #proof-of-usefulness #foundational-tech #developer-tools #cost-optimization

## The 3 a.m. page nobody wants

Picture the bug I keep running into. A pipeline asks an LLM for JSON. The prompt is strict. The schema is documented. The temperature is low. Everything looks fine in the notebook.

Then in production the model decides to be friendly. It sends back:

```
Sure, here's your JSON:
```json
{"answer": "Paris", "confidence": 0.95,}
```
```

Three things just broke my parser.

The triple backticks. The "Sure, here's your JSON" preamble. And one extra comma after `0.95`.

`serde_json::from_str` does what strict parsers do. It throws. My retry handler kicks in and fires the exact same prompt at the exact same model. Same input tokens. Same output tokens. Same cost. Same latency. The user waits another two seconds for what was already a correct answer.

I got tired of paying for that.

So I wrote `llm-json-repair`. It is a small Rust crate that does three boring things to a string before you hand it to a real parser. No model calls. No regex soup. No new dependencies on any LLM SDK. Just the three fixes I see break pipelines every week.

## What it is

`llm-json-repair` is on crates.io. One line to add it:

```toml
[dependencies]
llm-json-repair = "0.1"
```

The whole public surface is one function:

```rust
use llm_json_repair::repair;

let raw = "Sure, here you go:\n```json\n{\"answer\": \"Paris\", \"confidence\": 0.95,}\n```";
let cleaned = repair(raw);
assert_eq!(cleaned, r#"{"answer": "Paris", "confidence": 0.95}"#);
```

That is it. You pass the messy string. You get back a string that `serde_json` can actually parse.

If you turn on the optional `serde` feature, there is a one-shot helper that does the repair and the deserialize together:

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Answer {
    text: String,
    confidence: f64,
}

let raw = "```json\n{\"text\": \"hi\", \"confidence\": 0.5,}\n```";
let a: Answer = llm_json_repair::parse(raw)?;
```

I wanted the API to be impossible to misuse. There is no config struct. There are no flags. There is no policy. If you need policy, you do not need this crate.

## How it works

Three passes. Run in order. Each one fixes a real failure I have seen in real logs.

### Pass 1: strip the fences

LLMs love to wrap JSON in triple-backtick code fences. Sometimes with a language tag. Sometimes without. Sometimes the fence is on its own line. Sometimes it is inline.

Pass one finds the first triple backtick, walks past any language tag, and grabs everything up to the closing fence. If there is no fence, it skips this pass and passes the input through.

### Pass 2: extract the balanced JSON

Even after the fences come off, you often still have prose around the JSON. "Here is your answer:" before. "Let me know if you need anything else!" after. Some models add a polite closing paragraph no matter how many times you tell them not to.

Pass two walks the string until it hits the first `{` or `[`. Then it counts brackets, respecting strings and escape characters, until the structure closes. That balanced slice is what moves forward.

This is the pass that most homegrown cleaners get wrong. People reach for a regex. A regex cannot count balanced brackets. The first nested object eats it alive. A small hand-written walker is the right tool here.

### Pass 3: kill the trailing commas

JSON does not allow trailing commas. Most languages do. Most humans do. Most LLMs do.

Pass three scans for commas that sit immediately before `}` or `]` (allowing whitespace in between) and removes them. It is careful to leave commas inside strings alone.

After these three passes, the output is either valid JSON or a string that points you at a real bug. You hand it to `serde_json` and move on with your day.

### Rust-specific niceties

A few choices I made on purpose.

The only required dependency is `thiserror` for the error type. No `regex`. No `serde` unless you opt in. No tokio. No reqwest. No async. The crate compiles fast and adds nothing to your binary you did not already want.

It works in `no_std`-friendly contexts. There is no global state. No file I/O. No clock. You can call it from a Lambda cold start, from an edge worker, from a CLI on a Raspberry Pi.

The `serde` feature is gated behind a Cargo feature flag. If you want strict typing, turn it on. If you want raw strings, leave it off and save the compile time.

## Why this counts as foundational tech

Every team I have ever seen ship an LLM feature writes some version of this code. It usually lives in a file called `clean_json.py` or `parseModelOutput.ts`. It usually has a TODO at the top. It usually breaks the next time a model release changes its preamble style.

So the same three fixes get re-implemented across thousands of codebases. Slightly differently each time. Often with bugs. Often with regex that cannot handle nested objects.

A small shared library that does these three passes correctly, with tests, in a language fast enough to sit in front of any pipeline, removes that whole class of work from every team that picks it up.

The cost story is simple. If your pipeline does ten thousand model calls a day and one percent come back malformed, that is one hundred retries. At a few cents each, that is real money over a month. More importantly, every retry doubles the user-visible latency for that request. The model already knew the answer. You just could not read it.

Sitting a local repair pass in front of the parse step turns most of those retries into zero-cost local fixes. It works in front of any provider because it is a string-to-string function. Swap your model. Swap your SDK. Swap your cloud. The repair pass does not care.

That is what makes it foundational. It is small enough to be obvious. It composes with everything. It saves the second call that nobody should have to make.

## What is still missing

This is v0.1.0. I shipped it on May 9, 2026. I want to be honest about what it does not do yet.

It does not normalize smart quotes. If your model returns `"answer"` with curly quotes instead of straight quotes, this crate will not fix it. That one is on the roadmap because I have seen it from a few smaller fine-tuned models.

It does not strip comments. JSON does not allow `//` or `/* */` comments. Some models put them in anyway. Right now you get a parse error on those. Comment stripping is harder than it looks because you have to track string boundaries, but it is a real ask.

It does not repair broken unicode escapes inside strings. If a model emits `\u00` and stops, the string is malformed and this crate will not patch it. That kind of damage usually means the model truncated, and the right answer is to retry with more tokens, not to guess at the missing characters.

It does not do JSON5 or HJSON. Those are different formats with their own parsers. This crate stays inside spec JSON on purpose.

I would rather ship a small thing that always works than a big thing that sometimes does the wrong fix.

## Try it

The crate is on crates.io as `llm-json-repair`. Add it with one line in `Cargo.toml`.

The source is on GitHub at [github.com/MukundaKatta/llm-json-repair](https://github.com/MukundaKatta/llm-json-repair). The test suite has the examples I built it against. Issues and PRs are welcome, especially for repair cases I have not seen yet.

License is permissive so you can drop it into a commercial pipeline without a meeting.

If you ship anything that asks an LLM for JSON, this is the kind of code you already wrote once and would rather not write again. Use mine. Tell me what breaks.
