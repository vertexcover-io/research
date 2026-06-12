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

### Agent 5 — Adjacent concepts & vocabulary timeline (done)

- ReAct (Yao et al., Oct 2022, arxiv 2210.03629): loop lived *in the prompt*
  (Thought/Action/Observation). Modern while-loop practice drops the scaffolding
  because native tool-calling absorbed it; loop moved into ~10 lines of host code.
- Philipp Schmid "Inner Loop vs Outer Loop" (Feb 2026, philschmid.de): inner loop =
  ReAct-style within-task reliability; outer loop = across sessions, getting smarter
  over time (memory, skills, rules files). Cleanest published vocabulary split.
- Hamel Husain's loop = the eval/error-analysis feedback loop (improvement over time),
  distinct from the runtime agent loop.
- Vocabulary ownership timeline:
  - "tools in a loop" = Simon Willison ("An LLM agent runs tools in a loop to achieve
    a goal", settled Sep 2025 after crowdsourcing 211 definitions) + Anthropic
  - "agent = llm + memory + planning + tools + while loop" = swyx (Mar 2025)
  - "An agent is a LLM wrecking its environment in a loop" = Solomon Hykes (Jun 2025)
  - "own your control flow" (Factor 8) = Dex Horthy, 12-factor agents (Apr 2025)
  - "context engineering" = Anthropic (Sep 29, 2025) / LangChain
  - "harness engineering" = Mitchell Hashimoto (Feb 2026, "Agent = Model + Harness");
    legitimized by Latent Space AINews + AIE Europe 2026 Harness Engineering track;
    martinfowler.com article; HumanLayer "Skill Issue" post
  - "loop engineering" / "writing loops" = Cherny → Steinberger → Osmani (June 2026)
  - "compound engineering" = Every.to (Dan Shipper/Kieran Klaassen): plan → work →
    review → compound (lessons fed back so the next loop is better)
  - "Ralph loop" = Geoffrey Huntley (`while :; do cat PROMPT.md | claude-code; done`),
    Jul 2025; "everything is a ralph loop" ghuntley.com/loop/ Jan 2026
- Framework skepticism genre: Octomind "why we no longer use LangChain" (2024),
  Braintrust "canonical agent architecture: a while loop" (Aug 2025), dev.to "Agents
  are Loops". Counter-skepticism: AlphaSignal "Most Developers Do Not Need Agent
  Loops Yet" (cost critique, premature-completion failures).
- Caveat: Hashimoto coinage attribution is via secondary sources.
- Note: this agent said Cherny clip was "Acquired podcast" per one source — agent 1
  flagged that as likely conflation; dev-conf attribution better corroborated.

### Agent 3 — GitHub repos & code (done; code verified via raw.githubusercontent)

Canonical essays + code:
- Thorsten Ball "How to Build an Agent" (Apr 2025): Go, ~315 lines, "There is no
  moat." Community ports: leobeeson/single-file-ai-agent-tutorial (Python, 218-line
  main.py), ivanleomk/building-an-agent (TS), kevinyank.com JS port.
- sketch.dev 9-line loop (Philip Zeyliger, May 2025), verified verbatim:
  `while True: output, tool_calls = llm(msg); if tool_calls: msg = [handle_tool_call(tc)...] else: msg = user_input()`
  Single tool: bash. Crawshaw: "an agent is a for loop which contains an LLM call."
Anthropic's own:
- anthropics/claude-quickstarts agents/agent.py (~12-line _agent_loop), computer-use
  demo sampling_loop(); claude-cookbooks patterns/agents (45k stars).
- Agent SDK docs: "same tools, agent loop, and context management that power Claude
  Code"; Client SDK = you write the while loop, Agent SDK = Claude handles it.
Famous minimal harnesses:
- SWE-agent/mini-swe-agent (5.1k stars): "100 line AI agent", >74% SWE-bench
  verified, bash-only tool, run() = `while True: self.step()`.
- huggingface/smolagents (27.8k stars): ReAct loop in _run_stream, max_steps bound.
- HF Tiny Agents (Julien Chaumond): "an Agent is literally just a while loop on top
  of an MCP client" — Agent.ts in huggingface.js, 50-70 lines.
- simonw/llm 0.26 tools: chain() loop until no tool calls or chain_limit.
- humanlayer/12-factor-agents (23.2k stars): Factor 8 own-your-control-flow
  pseudocode loop.
Ralph family / loop-named repos:
- iannuttall/ralph (928 stars, TS): file-based agent loop, .ralph/ + PRD JSON, one
  story per fresh-context iteration, commits each pass, backend-agnostic AGENT_CMD.
- vercel-labs/ralph-loop-agent (795 stars, "Continuous Autonomy for the AI SDK"),
  coleam00/ralph-loop-quickstart (157), disler/infinite-agentic-loop (590).
- Micro-harnesses: genlayerlabs/subzeroclaw (C, ~380 lines), wedow/harness (bash+jq+
  curl), wulawulu/learn-claude-code-rs (Rust), bentossell/agent-loop (shell),
  sergenes/mini_agent (Python).
- The Register covered Ralph (Jan 2026): theregister.com/2026/01/27/ralph_wiggum_claude_loops/
