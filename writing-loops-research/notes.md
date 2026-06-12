# Notes: "Writing Loops" — agentic loop engineering research

## 2026-06-12 — Setup

Task: deep research on the concept of "writing loops" in agentic loop engineering.
- Find references (blogs, threads, talks, docs)
- Define the concept precisely
- How practitioners implement it (patterns, prompts, harnesses)
- Concrete GitHub repos / prompt examples
- Distinguish from adjacent concepts (ReAct, eval loops)

Plan: fan out 5 parallel web-research agents (the deep-research skill's Workflow
tool isn't available in this environment, so running the phases manually):

1. Origin & definition of "writing loops" / "loop engineering" terminology
2. Ralph Wiggum technique (Geoffrey Huntley) — the canonical "while loop" agent
3. Minimal agent-harness repos ("agent in a loop with tools")
4. Prompt/harness design patterns for long-running self-continuing loops
5. Adjacent concepts (ReAct, eval loops) + 2025–2026 discourse

Then: verify load-bearing claims, synthesize README.md with citations.

## Findings log

### Agent 1 — Origin & definition (done)

Key origin chain established:
- Dec 2024: Anthropic "Building Effective Agents" — "agents are typically just LLMs
  using tools based on environmental feedback in a loop."
- Apr 2025: Thorsten Ball, "How to Build an Agent" (ampcode.com) — "It's an LLM, a
  loop, and enough tokens." ~300-line Go agent.
- May 2025: David Crawshaw / sketch.dev "The Unreasonable Effectiveness of an LLM
  Agent Loop with Tool Use" — core loop is ~9 lines. HN: item?id=43998472
- Mid/late 2025: Geoffrey Huntley "Ralph Wiggum" bash while-loop (ghuntley.com/ralph)
- Sep 30, 2025: Simon Willison "Designing agentic loops" — loop design as a new skill
- ~June 5–6, 2026: **Boris Cherny (creator of Claude Code), on stage at Anthropic dev
  conf SF: "I don't prompt Claude anymore. I have loops that are running... My job is
  to write loops."** ← canonical origin of "writing loops" as a term. ~700K views/24h.
  He uninstalled his IDE; "a couple hundred agents read his GitHub, Slack, and Twitter
  and decide what to build next"; 259 PRs in a month all written by Claude Code.
- June 7–8, 2026: Peter Steinberger (@steipete) viral tweet: "you shouldn't be
  prompting coding agents anymore. You should be designing loops that prompt your
  agents." (status/2063697162748260627, 2.2M+ views)
- ~June 9, 2026: Addy Osmani "Loop Engineering" (addyosmani.com/blog/loop-engineering)
  — umbrella term: "Loop engineering is replacing yourself as the person who prompts
  the agent. You design the system that does it instead." Warns of "comprehension debt."
- Derivative wave (June 2026): Firecrawl, Pulumi ("A loop is a goal that prompts
  itself"), MindStudio ("a trigger and a verifiable goal"; stopping conditions as
  first-class), Medium posts with literal "writing loops" titles, Harness.io ("the
  intelligence is in the loop, not in the tools"), Braintrust ("canonical agent
  architecture: a while loop with tools"), Data Science Dojo (ReAct → loop
  engineering lineage), cobusgreyling/loop-engineering repo.
- Jeremy Howard (Sep 2025): "stop saying 'agent' and say 'tool loop' instead."
- Cherny three-stage progression: autocomplete → manually prompting 5–10 parallel
  sessions → writing loops that prompt Claude. Also: /loop = recurring agent jobs
  scheduled with cron (Peterman Pod, Dec 2025).
- Ralph became an official Claude Code plugin (anthropics/claude-code
  plugins/ralph-wiggum). History: humanlayer.dev/blog/brief-history-of-ralph.

Caveats: WebFetch egress-blocked for that agent — quotes via search snippets from
multiple independent sources. One source misattributed Cherny quote to Acquired
podcast; dev-conf attribution corroborated by 2+ sources.

Dead ends: swyx/Latent Space (no "writing loops" usage), Hamel Husain (no hits),
no Simon Willison comment on the June 2026 coinage yet.
