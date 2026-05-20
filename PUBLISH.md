# Publishing the 9 drafts to HackerNoon

I prepped 9 polished articles + 9 matching cover images, and got the **cachebench** draft 95 percent of the way to submission. The last 5 percent is uploading the featured image — HackerNoon enforces a sandboxed file picker that I can't drive programmatically.

## cachebench (already 95% done)

Editor URL: https://app.hackernoon.com/drafts/6a0785e2e3fb338333ad0620

Already set in the editor when you open it:
- Title: "cachebench: Find Out You Lost Your Prompt Cache Before the Bill Does"
- Body: full polished article (5,611 chars, all paragraphs rendered)
- Meta description: 135 chars, set
- TL;DR: 310 chars, set
- Tags: #proof-of-usefulness #llms #ai #open-source #ai-agents #observability #data
- Emoji Credibility Indicator: Guide

You need to do **2 things**:
1. Scroll to "CLICK TO UPLOAD FEATURED IMAGE" near the top → upload `covers/cachebench.png`
2. Click "Submit Story for Review!"

Done.

## The other 8 (per-draft 3-5 min flow)

| # | Project | Editor URL | Cover image to upload | Markdown to paste |
|---|---|---|---|---|
| 1 | driftvane | https://app.hackernoon.com/drafts/6a07833ae3fb338333ad05a2 (already created) | covers/driftvane.png | driftvane.md |
| 2 | agentsnap | (none yet — see below) | covers/agentsnap.png | agentsnap.md |
| 3 | agentguard | (none yet) | covers/agentguard.png | agentguard.md |
| 4 | agentcast | (none yet) | covers/agentcast.png | agentcast.md |
| 5 | agentvet | (none yet) | covers/agentvet.png | agentvet.md |
| 6 | agentmemory | (none yet) | covers/agentmemory.png | agentmemory.md |
| 7 | agenttrace | (none yet) | covers/agenttrace.png | agenttrace.md |
| 8 | bedrock-kit | (none yet) | covers/bedrock-kit.png | bedrock-kit.md |

### For the 7 "none yet" projects, create the draft first

Open the PoU report URL, **scroll to the bottom**, click **"Move Project into a HackerNoon Blog"**. Verify your email if asked (code goes to mukund.r1729@gmail.com). The button creates the draft on HackerNoon and gives you the editor URL.

PoU report URLs:
- agentsnap: https://proofofusefulness.com/reports/agentsnap-snapshot-tests-for-ai-agents
- agentguard: https://proofofusefulness.com/reports/agentguard-network-egress-firewall-for-ai-agents
- agentcast: https://proofofusefulness.com/reports/agentcast-structured-output-for-any-llm-call
- agentvet: https://proofofusefulness.com/reports/agentvet-validate-tool-args-before-execution
- agentmemory: open from your "60% ready" Gmail
- agenttrace: https://proofofusefulness.com/reports/agenttrace-cost-and-latency-tracking-for-agent-runs
- bedrock-kit: open from your "60% ready" Gmail

### Then in the editor for each one

1. Open the matching `.md` file in this repo
2. Replace the title with the first line (the `#` line — drop the leading `#`)
3. Select-all in the body editor, delete, paste from the .md (skip the title line)
4. Upload the matching `covers/<project>.png` as featured image
5. Set **Emoji Credibility Indicator** to "Guide" (or "Code License" — both fit)
6. Add a **Meta description** (first 1-2 sentences of the article, under 160 chars)
7. Add a **TL;DR** (2-3 sentences from the article)
8. Confirm tags include `#proof-of-usefulness` (required for the contest)
9. Set **Original on HackerNoon?** to "Yes"
10. Click **"Submit Story for Review!"**

## Why I couldn't finish this myself

Two hard gates:
1. The "Move Project into a HackerNoon Blog" button needs verifying an email code (one-time, already done for cachebench/driftvane, will need to re-do per session for the other 7)
2. The featured image upload is a sandboxed file picker — Chrome MCP only allows uploads from session-shared paths, and my generated covers fall outside that allowlist

Both are user-side manual steps that take seconds each. Total time for all 9: ~25-30 min.

## After all 9 submit successfully

HackerNoon editors review (typically 1-3 business days). Articles tagged `#proof-of-usefulness` automatically count toward the $150K prize pool. Score breakdowns from PoU (38 to 61.33) carry through.
