# Publishing the 9 drafts to HackerNoon

Each draft is a polished markdown article. Two of the 9 already have HackerNoon draft pages created by the PoU "Move Project" button. The other 7 need that button click first.

## Per-draft publish flow (~3-5 min each)

### 1. Open the HackerNoon editor for the draft

| Project | If draft exists, open this | Otherwise click "Move Project into a HackerNoon Blog" on |
|---|---|---|
| cachebench | https://app.hackernoon.com/drafts/6a0785e2e3fb338333ad0620 (already created) | — |
| driftvane | https://app.hackernoon.com/drafts/6a07833ae3fb338333ad05a2 (already created) | — |
| agentsnap | (none yet) | https://proofofusefulness.com/reports/agentsnap-snapshot-tests-for-ai-agents |
| agentguard | (none yet) | https://proofofusefulness.com/reports/agentguard-network-egress-firewall-for-ai-agents |
| agentcast | (none yet) | https://proofofusefulness.com/reports/agentcast-structured-output-for-any-llm-call |
| agentvet | (none yet) | https://proofofusefulness.com/reports/agentvet-validate-tool-args-before-execution |
| agentmemory | (none yet) | (visit the score email link) |
| agenttrace | (none yet) | https://proofofusefulness.com/reports/agenttrace-cost-and-latency-tracking-for-agent-runs |
| bedrock-kit | (none yet) | (visit the score email link) |

### 2. In the HackerNoon editor

1. Open the matching `.md` file in this repo
2. Replace the title with the first line of the .md (without the `#`)
3. Select all body in the editor, delete, paste the .md body
4. Required fields below the editor:
   - **Emoji Credibility Indicator** — pick one (e.g. 🛠️)
   - **Meta description** — first ~150 chars of the article hook
   - **TL;DR** — let HackerNoon's AI generate, or paste 2-3 sentences from the article
   - **Tags** — already prefilled, add #proof-of-usefulness if missing
   - **Original on HackerNoon?** — Yes
5. Click **Submit Story for Review!**

If submit returns a server error, the missing field is usually the Emoji Credibility Indicator. Pick one and retry.

## After all 9 are submitted

HackerNoon editors review and either publish or send back for edits. Published stories get tagged `#proof-of-usefulness` and count toward the contest leaderboard ($150K pool).

All 9 GitHub repos are public + working. The drafts are byte-faithful to the actual code.
