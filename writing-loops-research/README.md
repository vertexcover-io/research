# "Writing Loops" — Deep Research on the Agentic Loop-Engineering Concept

*Research date: 2026-06-12. Method: five parallel web-research agents (origin/definition, Ralph Wiggum deep-dive, GitHub harness repos, prompt/harness design patterns, adjacent vocabulary), followed by cross-agent verification of load-bearing claims. Code and prompt excerpts marked "verbatim" were fetched from the actual GitHub source files; quotes from fetch-blocked blogs come via multiple independent search extracts and are flagged where confidence is lower.*

---

## 1. TL;DR

**"Writing loops" is the practice of replacing yourself as the person who prompts a coding agent: instead of typing prompts interactively, you write an outer loop — a script, scheduler, or hook — that prompts the agent repeatedly, feeds back verification results, and runs until a measurable goal is met.**

The phrase as a named practice comes from **Boris Cherny (creator of Claude Code)**, on stage at Acquired Unplugged (presented by WorkOS) in San Francisco, June 2, 2026 ([video](https://www.youtube.com/watch?v=RkQQ7WEor7w)):

> "I don't prompt Claude anymore. I have loops that are running. They're the ones that are prompting Claude and figuring out what to do. **My job is to write loops.**"

The clip went viral (~700K views in 24h). Two days later **Peter Steinberger** amplified it ("you shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents" — 2M+ views), and ~June 9, 2026 **Addy Osmani** systematized it as **"loop engineering"** in a widely-cited post. But the underlying idea was fully developed in 2025 by Geoffrey Huntley's **Ralph Wiggum** bash loop, Thorsten Ball's 300-line agent, sketch.dev's 9-line loop, and Simon Willison's "designing agentic loops."

The one-line mental model, in its purest form (Huntley):

```bash
while :; do cat PROMPT.md | claude ; done
```

Everything else — specs, plan files, progress files, verification gates, stop conditions — is engineering *around* that loop.

---

## 2. What "writing loops" means, precisely

There are actually **two loops**, and the term shift in 2026 is about which one you engineer:

| | Inner loop | Outer loop |
|---|---|---|
| What it is | LLM ↔ tool-calls until no more tool calls | Script/scheduler that re-prompts the agent across sessions |
| Who writes it | The vendor (Claude Code, Codex) or ~10 lines of your code | **You — this is "writing loops"** |
| State | The context window | Files, git, task queues |
| Lineage | ReAct (2022) → native tool-calling | Ralph (2025) → Cherny/Osmani (2026) |
| Vocabulary | "tools in a loop", "agent loop" | "writing loops", "loop engineering", "Ralph loop" |

Philipp Schmid published the cleanest version of this split as ["Agents: Inner Loop vs Outer Loop"](https://www.philschmid.de/inner-loop-vs-outer-loop) (Feb 2026): the inner loop is "about reliability within a task"; the outer loop is "about getting smarter over time."

Definitions from the primary sources:

- **Addy Osmani** ([Loop Engineering](https://addyosmani.com/blog/loop-engineering/), June 2026): "Loop engineering is replacing yourself as the person who prompts the agent. You design the system that does it instead."
- **Pulumi** ([Stop Prompting. Design the Loop.](https://www.pulumi.com/blog/stop-prompting-design-the-loop/)): "A loop is a goal that prompts itself. You set the purpose, and the system keeps iterating until it's met."
- **MindStudio** ([What Is Loop Engineering?](https://www.mindstudio.ai/blog/what-is-loop-engineering-ai-coding-agents)): a loop needs "a trigger and a verifiable goal," and "good loop engineering treats stopping conditions as first-class design requirements."
- **Firecrawl** ([Loop Engineering](https://www.firecrawl.dev/blog/loop-engineering)): "Prompt engineering optimizes a single turn… Loop engineering optimizes a system that runs many turns without you. The loop is only as good as the feedback inside it."

Cherny's implied three-stage progression of the practitioner ([OfficeChai coverage](https://officechai.com/ai/i-now-just-write-loops-to-prompt-claude-code-claude-code-creator-boris-cherny/)):

1. Hand-writing code with autocomplete
2. Manually prompting 5–10 parallel agent sessions
3. **Writing loops that prompt the agents** — Cherny reports he uninstalled his IDE, runs "a couple hundred agents" that read his GitHub/Slack/Twitter to decide what to build, and shipped 259 PRs in a month with every line written by Claude Code.

---

## 3. Reference timeline (all notable sources)

### Conceptual precursors — "the agent IS a loop" (2022–2025)

| Date | Source | Contribution |
|---|---|---|
| Oct 2022 | [ReAct paper](https://arxiv.org/abs/2210.03629) (Yao et al.) | Thought→Action→Observation cycle — the loop lived *in the prompt* |
| Dec 2024 | [Anthropic, "Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) (Schluntz & Zhang) | "Agents are typically just LLMs using tools based on environmental feedback in a loop"; workflows-vs-agents; anti-framework stance |
| Mar 2025 | [swyx, "Agent Engineering"](https://www.latent.space/p/agent) | agent = llm + memory + planning + tools + **while loop** |
| Apr 2025 | [Thorsten Ball, "How to Build an Agent"](https://ampcode.com/how-to-build-an-agent) | "It's an LLM, a loop, and enough tokens" — working code-editing agent in ~315 lines of Go. "There is no moat." |
| Apr 2025 | [Dex Horthy, 12-Factor Agents](https://github.com/humanlayer/12-factor-agents) | Factor 8: "Own your control flow" — write the loop yourself, don't let a framework own it |
| May 2025 | [sketch.dev, "The Unreasonable Effectiveness of an LLM Agent Loop"](https://sketch.dev/blog/agent-loop) (Zeyliger) | The famous 9-line Python loop with one tool (bash). [HN](https://news.ycombinator.com/item?id=43998472) |
| Jun 2025 | Solomon Hykes, AI Engineer keynote | "An agent is an LLM wrecking its environment in a loop" |
| Jul 2025 | [Geoffrey Huntley, "Ralph Wiggum as a 'software engineer'"](https://ghuntley.com/ralph/) | **The literal outer loop**: `while :; do cat PROMPT.md \| claude ; done`. [HN](https://news.ycombinator.com/item?id=44565028) |
| Sep 2025 | [Simon Willison, "Agents"](https://simonwillison.net/2025/Sep/18/agents/) + Jeremy Howard ["tool loop"](https://x.com/jeremyphoward/status/1968771130304434502) | Settled definition: "An LLM agent runs tools in a loop to achieve a goal" (from 211 crowdsourced definitions) |
| Sep 30, 2025 | [Simon Willison, "Designing agentic loops"](https://simonwillison.net/2025/Sep/30/designing-agentic-loops/) | Loop design named as a *skill*: "carefully selecting tools to run in a loop to achieve a specified goal. Do this well and you can solve many coding problems with brute force." |
| Aug 2025 | [Braintrust, "The canonical agent architecture: a while loop with tools"](https://www.braintrust.dev/blog/agent-while-loop) | Every major product (Claude Code, Codex, Cursor, LangGraph, smolagents) converges on the same while-loop |
| Nov 2025 | [Anthropic, "Effective harnesses for long-running agents"](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | Anthropic's own outer-loop recipe: initializer + per-session coding agent + progress files (see §5) |

### The naming moment (June 2026)

| Date | Source | Contribution |
|---|---|---|
| Jun 2, 2026 | **Boris Cherny on stage, Acquired Unplugged (WorkOS), SF** ([video](https://www.youtube.com/watch?v=RkQQ7WEor7w), [clip](https://x.com/Av1dlive/status/2063592868581978517), [coverage](https://officechai.com/ai/i-now-just-write-loops-to-prompt-claude-code-claude-code-creator-boris-cherny/)) | "My job is to write loops" — origin of the term |
| Jun 7–8, 2026 | [Peter Steinberger tweet](https://x.com/steipete/status/2063697162748260627) | "You should be designing loops that prompt your agents" — the viral vector (2M+ views) |
| ~Jun 9, 2026 | [Addy Osmani, "Loop Engineering"](https://addyosmani.com/blog/loop-engineering/) | Names and systematizes the practice; warns of **"comprehension debt"**: "the faster the loop ships code you did not write, the larger the distance between what the repository contains and what you understand." |
| Jun 2026 | Derivative wave | [Firecrawl](https://www.firecrawl.dev/blog/loop-engineering), [Pulumi](https://www.pulumi.com/blog/stop-prompting-design-the-loop/), [MindStudio](https://www.mindstudio.ai/blog/what-is-loop-engineering-ai-coding-agents), [Harness.io](https://www.harness.io/blog/agent-loop-new-os) ("the intelligence is in the loop, not in the tools"), Medium posts titled literally ["Stop Prompting Your Agent. Start Writing Loops."](https://medium.com/@garbarok/stop-prompting-your-agent-start-writing-loops-73608223f075) and ["I Don't Prompt Claude Anymore. I Write Loops That Prompt Claude."](https://medium.com/@fahey_james/i-dont-prompt-claude-anymore-i-write-loops-that-prompt-claude-57e48a4f28d7), [Data Science Dojo ReAct→loop-engineering guide](https://datasciencedojo.com/blog/agentic-loops-explained-from-react-to-loop-engineering-2026-guide/), [cobusgreyling/loop-engineering](https://github.com/cobusgreyling/loop-engineering) |

Related earlier Cherny material: [Peterman Pod interview, Dec 2025](https://youtu.be/AmdLVWMdjOk) (parallel sessions, sub-agents, `/loop` as "recurring agent jobs scheduled with cron"); [Pragmatic Engineer, "How Claude Code is built"](https://newsletter.pragmaticengineer.com/p/how-claude-code-is-built). *(Note: one secondary source attributed the stage quote to the Acquired podcast — multiple other sources corroborate the dev-conference attribution; treat the exact venue as medium-confidence, the quote text as high-confidence.)*

---

## 4. The Ralph Wiggum technique — the canonical written loop

Geoffrey Huntley's **Ralph** ([ghuntley.com/ralph](https://ghuntley.com/ralph/), July 2025) is the purest pre-2026 embodiment of writing loops and the direct ancestor of the practice:

```bash
# original (with Sourcegraph amp):
while :; do cat PROMPT.md | npx --yes @sourcegraph/amp ; done
# with Claude Code:
while :; do cat PROMPT.md | claude ; done
```

Key ideas, in Huntley's words: "Ralph is a technique. In its purest form, Ralph is a Bash loop." It is "deterministically bad in a nondeterministic world" — its failures are predictable, so you "tune it like a guitar," adding guardrail "signs" to the prompt as you observe failure modes. Famous result: a $50K USD contract delivered as a tested MVP for **$297 in API costs**; a 3-month loop produced ["cursed"](https://ghuntley.com/cursed/), a Gen-Z-slang programming language with an LLVM backend ([PC Gamer coverage](https://www.pcgamer.com/software/ai/slay-bestie-just-some-of-the-gen-z-slang-used-in-programming-language-dreamt-up-by-ai-in-a-ralph-wiggum-loop-that-its-creator-feared-may-be-too-powerful/)). Caveat from the same post: "Engineers are still needed. There is no way this is possible without senior expertise guiding Ralph."

**Why it works** (per Huntley's [allocations](https://ghuntley.com/allocations/)/[backpressure](https://ghuntley.com/pressure/) posts, [HumanLayer's "Brief History of Ralph"](https://www.humanlayer.dev/blog/brief-history-of-ralph), and [Steve Kinney's explainer](https://stevekinney.com/writing/the-ralph-loop)):

- **Fresh context per iteration defeats context rot.** HumanLayer's empirical "dumb zone" finding: past ~40% context utilization, "signal-to-noise degrades, attention fragments, and agents start making mistakes." Ralph restarts before the rot.
- **Files and git are the memory layer, not the context window.** (Kinney: "your files and git history are a better memory layer than the LLM's context window.")
- **Backpressure**: tests, typecheckers, and linters reject bad generations before they're committed; "the LLM isn't protected from its own mess; it is forced to confront it" (Horthy).
- **"Let Ralph Ralph"** — "your job is to sit on the loop, not in it."

**The playbook** ([ghuntley/how-to-ralph-wiggum](https://github.com/ghuntley/how-to-ralph-wiggum), with [ClaytonFarr/ralph-playbook](https://github.com/ClaytonFarr/ralph-playbook)): "**3 Phases, 2 Prompts, 1 Loop**" —

1. **Specs**: a conversation produces `specs/*.md` (one per Jobs-To-Be-Done topic)
2. **Planning loop** (`PROMPT_plan.md`): gap analysis between specs and code → `IMPLEMENTATION_PLAN.md`. "Plan only. Do NOT implement. Do NOT assume functionality is missing; confirm with code search first."
3. **Building loop** (`PROMPT_build.md`): pick the ONE most important item, implement completely, run tests, commit, update the plan, exit; the bash loop restarts it.

The production loop (verbatim from the playbook's `loop.sh`): `cat "$PROMPT_FILE" | claude -p --dangerously-skip-permissions --output-format=stream-json --model opus --verbose` inside `while true`, with max-iterations and a `git push` per iteration. The build prompt carries Huntley's signature quirks: orientation steps `0a/0b/0c` ("Study `specs/*` with up to 500 parallel Sonnet subagents"), "only 1 subagent for build/tests" (backpressure throttle), and escalating-9s guardrails ("999999999999. Implement functionality completely. Placeholders and stubs waste efforts").

**Official adoption & the fidelity controversy**: Anthropic shipped Ralph as an official Claude Code plugin ([plugins/ralph-wiggum](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum)) — `/ralph-loop "task" --completion-promise "DONE" --max-iterations 50` — but implemented it with a **Stop hook that re-feeds the prompt within one session**, so context *accumulates* rather than resetting. [Issue #125](https://github.com/anthropics/claude-plugins-official/issues/125) ("Plugin deviates from original Ralph behavior") and a Huntley/Horthy video (["Ralph Wiggum (and why Claude Code's implementation isn't it)"](https://www.youtube.com/watch?v=O2bBWDoxO4s)) document the disagreement — which is itself the central design debate in loop writing (see §5, "two continuation philosophies").

**Critiques**: [Pedro Nauck's "Ralph Wiggum Doesn't Work"](https://news.ycombinator.com/item?id=46672413) (cascade failures in structured work: "If Task 1 fails catastrophically, Task 2 starts anyway... a Jenga tower of broken code by Task 10"); HN reviewers on the original post found output quality "far below" expectations; [codecentric's experience report](https://www.codecentric.de/en/knowledge-hub/blog/the-ralph-wiggum-loop-autonomous-code-generation-with-a-fresh-context) names the two failure modes **"overcooking"** (loop never stops, keeps adding unrequested features) and **"undercooking"** (stops too early); [AlphaSignal](https://alphasignalai.substack.com/p/most-developers-do-not-need-agent) adds the cost critique ("loops favor whoever can spend"). Press: [VentureBeat](https://venturebeat.com/technology/how-ralph-wiggum-went-from-the-simpsons-to-the-biggest-name-in-ai-right-now), [The Register](https://www.theregister.com/2026/01/27/ralph_wiggum_claude_loops/).

---

## 5. How to do it — the consensus playbook

Synthesized from Anthropic's harness posts + quickstart prompts, the Ralph playbook, OpenAI's Codex long-horizon guidance, obra/superpowers, and community harnesses. The loop has four parts: **a dumb script, a smart prompt, persistent state on disk, and verification gates** (Kinney's framing: "bash script = dumbest part, PROMPT.md = the brain, filesystem = memory, backpressure = verification").

### 5.1 The prompt structure (what goes in PROMPT.md)

Consensus iteration-prompt skeleton, with the source of each element:

1. **Fresh-context identity** — tell the model it's one iteration of many. Anthropic's [autonomous-coding quickstart](https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding) opens verbatim: *"You are continuing work on a long-running autonomous development task. This is a FRESH context window - you have no memory of previous sessions."*
2. **Mandatory orientation ritual** — read progress file, plan, spec, `git log`, count remaining work. (Anthropic step 1 "GET YOUR BEARINGS (MANDATORY)"; Ralph's steps `0a/0b/0c`.)
3. **Regression check before new work** — "The previous session may have introduced bugs"; re-run 1–2 passing tests and a smoke test first.
4. **Scope clamp: ONE task per iteration** — universal across Ralph, Anthropic, cwc, superpowers. Explicitly absolve the model: *"It's ok if you only complete one feature in this session, as there will be more sessions later."*
5. **Search before assuming** — "Don't assume not implemented; confirm with code search first" (Ralph calls assuming-not-implemented the technique's Achilles' heel — it causes duplicate implementations).
6. **Verification with anti-shortcut rules** — real browser automation, screenshots, evidence files; "Only testing with curl... is insufficient."
7. **End-of-iteration ritual** — update plan/progress file ("what should be worked on next"), descriptive git commit, leave the tree green.
8. **Guardrails** — accumulated reactively as you observe failures (Ralph's ascending-9s; "no placeholders/stubs"; append-only task lists: *"IT IS CATASTROPHIC TO REMOVE OR EDIT FEATURES IN FUTURE SESSIONS"*).
9. **Continuous-execution clause** (for in-session loops) — obra/superpowers verbatim: *"Do not pause to check in with your human partner between tasks... The only reasons to stop are: BLOCKED status you cannot resolve, ambiguity that genuinely prevents progress, or all tasks complete."*

### 5.2 The state that persists between iterations (the loop's memory)

| File | Role | Rules |
|---|---|---|
| `feature_list.json` / `test-results.json` / `prd.json` | Goal state + progress counter | Append-only goals; the agent may flip only the `passes` flag; **default-FAIL** (everything starts false) |
| `IMPLEMENTATION_PLAN.md` / `PROGRESS.md` / `claude-progress.txt` | Handoff note between iterations | Curated state ("Done / In progress / Next / Notes"), *not* a transcript — Anthropic's Pokémon harness showed transcript-style memory produces 31 junk files and no progress |
| `AGENTS.md` / `CLAUDE.md` | Operational how-to (build/test commands) | Keep lean — "a bloated AGENTS.md pollutes every future loop's context" |
| `specs/*.md` | Immutable source of truth | Written before the loop starts |
| git history | Second memory + rollback | Commit every iteration; descriptive messages; agents use `git revert` to escape broken states |
| `current_tasks/*.lock` | Parallel-loop coordination | Carlini's C-compiler harness: agents claim tasks via lock files synced through git — no orchestrator needed |

### 5.3 Stop conditions (the menu observed in the wild)

- **Work queue empty** — done is data, not vibes: `while grep -q '"passes": false' test-results.json; do ... done` ([anthropics/cwc-long-running-agents](https://github.com/anthropics/cwc-long-running-agents), verbatim)
- **Completion promise** — exact-match magic string the model may emit only when true: `<promise>DONE</promise>` "(ONLY when statement is TRUE - do not lie to exit!)" (ralph-wiggum plugin)
- **Independent evaluator** — a fresh-context, no-write subagent grades the work PASS/NEEDS_WORK; "the builder shouldn't grade its own work"; on NEEDS_WORK its findings become the next iteration's starting prompt. Claude Code's built-in `/goal` productizes this (a fast model checks the completion condition after every turn). Cherny's rule: *"always give Claude a way to verify its work... you can't manually review a 26-hour thread — the system must verify itself."*
- **Hard budgets** — max iterations / cost / wall-clock; universally recommended as the safety net
- **Stall detection** — "a cycle makes no changes"; [frankbria/ralph-claude-code](https://github.com/frankbria/ralph-claude-code)'s dual-condition exit (completion indicators AND explicit `EXIT_SIGNAL: true`) plus circuit breakers (`MAX_CONSECUTIVE_TEST_LOOPS=3`, consecutive-done-signal limits)
- **Human escape hatches** — `touch AGENT_STOP` kill-switch; `STEER.md` mid-run injection ("OPERATOR STEERING: ...higher priority than your current plan")

### 5.4 The central design choice: two continuation philosophies

1. **Stateless restart (Ralph-style)**: fresh process per iteration, memory on disk. Exploits the fresh-context "smart zone" (40–60% utilization); favored by Huntley, the Nov-2025 Anthropic harness, snarktank/ralph.
2. **One long session with compaction (Codex-style)**: OpenAI's `/responses/compact` endpoint; Anthropic's Mar-2026 update reports that **with Opus 4.5 they dropped context resets entirely** in favor of one continuous session with Agent SDK auto-compaction.

Anthropic's own meta-lesson ([Harness design for long-running apps](https://www.anthropic.com/engineering/harness-design-long-running-apps), Mar 2026): *"every component in a harness encodes an assumption about what the model can't do on its own... worth stress testing... they can quickly go stale."* The loop you write today is scaffolding for today's model limitations.

---

## 6. GitHub repos & code examples

### The inner loop, minimal (write-your-own-agent)

| Repo / source | Language | The loop |
|---|---|---|
| [sketch.dev agent-loop post](https://sketch.dev/blog/agent-loop) ([mirror](https://blog.philz.dev/blog/agent-loop/)) | Python | The famous 9 lines: `while True: output, tool_calls = llm(msg); if tool_calls: msg = [handle_tool_call(tc) for tc in tool_calls] else: msg = user_input()` — single tool: bash |
| [Thorsten Ball, "How to Build an Agent"](https://ampcode.com/how-to-build-an-agent) | Go, ~315 lines | `for { getUserMessage → runInference → dispatch tool_use blocks }`; ports: [leobeeson/single-file-ai-agent-tutorial](https://github.com/leobeeson/single-file-ai-agent-tutorial) (Python, 218-line main.py), [ivanleomk/building-an-agent](https://github.com/ivanleomk/building-an-agent) (TS), [kevinyank.com JS port](https://kevinyank.com/posts/how-to-build-an-agent-in-javascript/) |
| [anthropics/claude-quickstarts](https://github.com/anthropics/claude-quickstarts) `agents/agent.py` | Python | ~12-line `_agent_loop()`: create message → execute tool calls → append results → repeat until no tool calls. Also the computer-use demo's `sampling_loop()` |
| [SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) (~5.1k★) | Python | "The 100 line AI agent that solves GitHub issues" — >74% on SWE-bench Verified; `run()` = `while True: self.step()`; bash is the only tool |
| [huggingface/smolagents](https://github.com/huggingface/smolagents) (~27.8k★) | Python | ReAct loop in `_run_stream`, bounded by `max_steps` |
| [HF Tiny Agents](https://huggingface.co/blog/tiny-agents) ([Agent.ts in huggingface.js](https://github.com/huggingface/huggingface.js)) | TS/Python, ~50–70 lines | "An Agent is literally just a while loop on top of an MCP client" |
| [simonw/llm](https://github.com/simonw/llm) (~12k★) | Python | `Conversation.chain()` executes tool calls and re-prompts until none remain or `chain_limit` |
| [humanlayer/12-factor-agents](https://github.com/humanlayer/12-factor-agents) (~23.2k★) | pseudocode | Factor 8 loop: `while True: next_step = llm.determine_next_step(context); if done: return; context.append(execute_step(next_step))` |
| Micro-harnesses | C / Bash / Rust | [genlayerlabs/subzeroclaw](https://github.com/genlayerlabs/subzeroclaw) (~380 lines of C, 54KB binary), [wedow/harness](https://github.com/wedow/harness) (bash + jq + curl), [wulawulu/learn-claude-code-rs](https://github.com/wulawulu/learn-claude-code-rs), [bentossell/agent-loop](https://github.com/bentossell/agent-loop), [sergenes/mini_agent](https://github.com/sergenes/mini_agent) |

### The outer loop (the "writing loops" repos)

| Repo | What it adds |
|---|---|
| [ghuntley/how-to-ralph-wiggum](https://github.com/ghuntley/how-to-ralph-wiggum) + [ClaytonFarr/ralph-playbook](https://github.com/ClaytonFarr/ralph-playbook) | The canonical playbook: `loop.sh`, `PROMPT_plan.md`, `PROMPT_build.md`, specs/plan file conventions (§5) |
| [anthropics/claude-quickstarts → autonomous-coding](https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding) | Anthropic's own outer-loop prompts verbatim: initializer + per-session coding prompt, `feature_list.json` contract |
| [anthropics/cwc-long-running-agents](https://github.com/anthropics/cwc-long-running-agents) | Three harness primitives: default-FAIL contract enforced by a PreToolUse hook, fresh-context evaluator subagent, agent-maintained PROGRESS.md handoff; kill-switch + steering files; minimal `while grep -q '"passes": false'` outer loop |
| [anthropics/claude-code → plugins/ralph-wiggum](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum) | Official plugin: Stop-hook re-prompting, `--completion-promise`, `--max-iterations` (note the fresh-context controversy, §4) |
| [snarktank/ralph](https://github.com/snarktank/ralph) (Ryan Carson) | PRD-driven: `prd.json` + `progress.txt` + git as memory; "Each iteration is a fresh instance with clean context"; works with Amp or Claude Code |
| [iannuttall/ralph](https://github.com/iannuttall/ralph) (~0.9k★) | "A minimal, file-based agent loop for autonomous coding" — backend-agnostic (`AGENT_CMD`: claude/codex/droid/opencode), one story per iteration |
| [vercel-labs/ralph-loop-agent](https://github.com/vercel-labs/ralph-loop-agent) (~0.8k★) | Ralph as a TypeScript AI-SDK library with verification callbacks |
| [mikeyobrien/ralph-orchestrator](https://github.com/mikeyobrien/ralph-orchestrator) | Rust orchestrator, 7 AI backends, "Hat" personas, TUI |
| [frankbria/ralph-claude-code](https://github.com/frankbria/ralph-claude-code) | Exit detection + circuit breakers (§5.3) |
| [disler/infinite-agentic-loop](https://github.com/disler/infinite-agentic-loop) (~0.6k★) | Two-prompt infinite generation pattern for Claude Code |
| [obra/superpowers](https://github.com/obra/superpowers) (Jesse Vincent) | Plan-driven loop via skills: fresh subagent per task + two-stage review (spec compliance, then code quality); continuous-execution clause |
| [coleam00/ralph-loop-quickstart](https://github.com/coleam00/ralph-loop-quickstart), [Th0rgal/open-ralph-wiggum](https://github.com/Th0rgal/open-ralph-wiggum), [agrimsingh/ralph-wiggum-cursor](https://github.com/agrimsingh/ralph-wiggum-cursor), [Block Goose ralph tutorial](https://block.github.io/goose/docs/tutorials/ralph-loop/) | Ecosystem variants; index: [snwfdhmp/awesome-ralph](https://github.com/snwfdhmp/awesome-ralph) |

### Scale demonstration

[Anthropic / Nicholas Carlini, "Building a C compiler with a team of parallel Claudes"](https://www.anthropic.com/engineering/building-c-compiler) (Feb 2026): a harness "that sticks Claude in a simple loop where when it finishes one task, it immediately picks up the next" — **16 parallel agents, ~2,000 fresh sessions, $20K in API costs, a 100K-line Rust C compiler that builds Linux 6.9**. Coordination is filesystem + git lock files, no orchestrator; correctness comes from a massive compiled-test backpressure suite. ([Register](https://www.theregister.com/2026/02/09/claude_opus_46_compiler/), [InfoQ](https://www.infoq.com/news/2026/02/claude-built-c-compiler/))

---

## 7. Distinguishing "writing loops" from adjacent concepts

| Term | Owner / origin | What it names | Relation |
|---|---|---|---|
| **ReAct loop** | Yao et al., Oct 2022 | Thought→Action→Observation *prompting pattern* | Ancestor of the inner loop; native tool-calling APIs absorbed the scaffolding, moving the loop from the prompt into ~10 lines of host code |
| **"Tools in a loop" / agent loop** | Willison (Sep 2025), Anthropic | The *inner* loop: model ↔ tools until done | What vendors ship; writing loops sits *around* it |
| **Eval loop / feedback loop** | Hamel Husain et al. | The *development-time* improvement cycle: error analysis → annotation → judges → iterate | Improves the agent over releases; the written loop runs the agent within a task. The cwc evaluator subagent is an eval loop embedded *inside* a written loop |
| **Context engineering** | Anthropic (Sep 2025) / LangChain | Managing what's in the window: compaction, note-taking, sub-agents | The discipline that makes loop iterations work; fresh-context looping is a context-engineering move |
| **Harness engineering** | Mitchell Hashimoto (Feb 2026, "Agent = Model + Harness"; attribution via secondary sources) → [Latent Space](https://www.latent.space/p/ainews-is-harness-engineering-real), [martinfowler.com](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html), [HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents) | The whole environment around the model (tools, hooks, permissions, AGENTS.md) | Superset: the loop is one harness component. AIE Europe 2026 ran the first "Harness Engineering" track |
| **Own your control flow** | Dex Horthy, 12-factor agents (Apr 2025) | Anti-framework principle: implement the loop in your code | The architectural argument that legitimized writing your own loop |
| **Compound engineering** | Every.to (Shipper/Klaassen) | plan → work → review → **compound** (feed lessons back so the next loop is better) | The outer-outer loop: improving the loop itself over time |
| **Inner vs outer loop** | Philipp Schmid (Feb 2026) | The two-loop vocabulary itself | "Writing loops" ≈ engineering the outer loop |

---

## 8. Assessment

**What's genuinely new:** nothing mechanical — `while true` is not an invention. What changed in mid-2026 is the *identity claim*: the engineer's primary artifact shifts from prompts to loop systems (trigger + goal contract + verification + stop conditions), with Cherny as the existence proof. The load-bearing engineering is all in §5: default-FAIL goal contracts, curated handoff files, independent verification, and explicit stop conditions. "The loop is only as good as the feedback inside it" (Firecrawl) is the one-line summary of every source.

**What to be skeptical of:** (1) survivor bias — the viral demos (cursed, the C compiler) had enormous test-suite backpressure; loops without verifiable goals wander ("overcooking"/"undercooking"); (2) cost — Ralph burns ~$10/hr; Carlini's compiler cost $20K; (3) comprehension debt (Osmani) — the repo outruns your understanding of it; (4) staleness — Anthropic deleted its own fresh-context machinery within four months as models improved; the loop you write is scaffolding for the current model generation, not architecture; (5) structured multi-step work still fails in naive loops (Nauck's cascade-failure critique) — hence the trend toward plan files, task isolation, and evaluator gates.

**Confidence notes:** all code/prompt excerpts marked verbatim were fetched from GitHub sources. The Cherny quote text is corroborated by multiple independent sources; follow-up research (§9) settled the venue as **Acquired Unplugged, presented by WorkOS, SF, June 2, 2026**. The Hashimoto "harness engineering" coinage attribution is via secondary sources. View counts are approximate and inconsistent across sources.

---

## 9. Addendum: the actual loops Cherny and Steinberger run (the standing kind)

*Follow-up investigation. The Ralph/`/goal` loop (§4–5) iterates on ONE task until done. The loops Cherny and Steinberger were referring to are different: **standing loops** — scheduled or event-triggered processes that ORIGINATE work each tick (read inputs, decide what to do, prompt the agent, ship a PR), running indefinitely. "A loop is cron plus a decision-maker."*

### 9.1 Boris Cherny's loops (verified, his own posts)

His actual running loop list, from his ["15 underrated Claude Code features" thread](https://x.com/bcherny/status/2038454341884154269) (Mar 30, 2026, archived in [GitHub mirrors](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-boris-15-tips-30-mar-26.md)):

```
/loop 5m  /babysit            # auto-address code review, auto-rebase, shepherd PRs to production
/loop 30m /slack-feedback     # automatically put up PRs for Slack feedback every 30 mins
/loop     /post-merge-sweeper # put up PRs to address code review comments I missed
/loop 1h  /pr-pruner          # close out stale and no-longer-necessary PRs
# "...and lots more!"
```

His stated pattern: **"Experiment with turning workflows into skills + loops."** Each loop = a custom skill (the standing prompt/intent) + `/loop <interval>` (the trigger). The skill bodies were never published; public versions are community reconstructions. Launch post for `/loop` (~Mar 7, 2026): *"/loop is a powerful new way to schedule recurring tasks, for up to 3 days at a time. eg. '/loop babysit all my PRs. Auto-fix build issues and when comments come in, use a worktree agent to fix them'."*

**What `/loop` actually is** ([official docs](https://code.claude.com/docs/en/scheduled-tasks)): a bundled skill (Claude Code ≥2.1.72) on a real in-session cron scheduler — internal `CronCreate`/`CronList`/`CronDelete` tools, ≤50 tasks/session, fires only when the session is idle, jitter ≤30 min, 7-day expiry. With no interval it **self-paces** (picks a 1min–1h delay each tick based on what it observed). With no prompt it runs a built-in maintenance prompt (tend the current PR: review comments, failed CI, conflicts), replaceable via `.claude/loop.md`. The escalation ladder: `/loop` (in-session) → [Desktop scheduled tasks](https://code.claude.com/docs/en/desktop-scheduled-tasks.md) (fresh session per run, prompt stored as `~/.claude/scheduled-tasks/<name>/SKILL.md`, can self-reschedule) → [Routines](https://code.claude.com/docs/en/routines.md) (Anthropic-managed cloud; triggers = cron ≥1h, per-routine API webhook, or GitHub events with filters; pushes only to `claude/`-prefixed branches). The "couple hundred agents" come from **dynamic workflows** (v2.1.154: Claude orchestrates tens–hundreds of agents in the background) plus `/batch` fan-out over git worktrees.

Scale claims (verified via [Simon Willison](https://simonwillison.net/2025/Dec/27/boris-cherny/) and [Fortune, Jun 8, 2026](https://fortune.com/2026/06/08/anthropics-boris-cherny-creator-of-claude-code-says-there-are-days-he-manages-tens-of-thousands-of-ai-agents-at-once/)): 259 PRs / 497 commits / 40k lines added in 30 days, every line Claude-written; "a few hundred [agents]... some days... tens of thousands"; no handwritten code in ~8 months. Secondhand retellings add a CI-flaky-test-patcher loop and a Twitter-feedback-clustering loop every 30 min.

### 9.2 Peter Steinberger's loops (verified, code on GitHub)

His June 2026 post was a "monthly reminder" of a doctrine he ships as code:

- **[autoreview skill](https://github.com/openclaw/agent-skills/blob/main/skills/autoreview/SKILL.md)** (in [steipete/agent-scripts](https://github.com/steipete/agent-scripts)): *"runs codex /review in a loop until there's no booboos anymore."* Findings are advisory — the agent verifies each independently, re-runs tests + review after accepting fixes; the loop ends when no accepted findings remain. The stop condition, not the review call, is the engineered part.
- **maintainer-orchestrator skill** (same repo): a control-plane loop over his repos — classifies queue items (autonomous / owner-decision / ignore), delegates to workers, monitors them every 5 minutes; worker contract requires regression tests + "execute autoreview until findings are resolved" + a live-proof gate; release only at zero open issues + green CI.
- **[ClawSweeper](https://github.com/openclaw/clawsweeper)**: the production fleet loop over OpenClaw's 7,000+ issue backlog. Four lanes — Review (proposes closes, never applies), Apply (wakes every 15 min, executes only unchanged high-confidence proposals), Repair (`@clawsweeper autofix` → bounded AI fix loop where "deterministic executor steps own every GitHub mutation"), Commit Review. Safety: **"AI runs without GitHub write tokens"**; adaptive cadence (hot items hourly, <30 days daily, older weekly).
- **OpenClaw's loop primitives** ([automation docs](https://docs.openclaw.ai/automation)): Cron (SQLite-persisted, isolated `cron:<jobId>` sessions, retry backoff), **Heartbeat** (a main-session agent turn every 30 min reading a `HEARTBEAT.md` checklist; replying `HEARTBEAT_OK` is suppressed — the engineered silence condition), Background tasks, Task Flow, **Standing Orders** ("define programs with clear scope, triggers, and escalation rules — and the agent executes autonomously within those boundaries"), Hooks. Decision rule: cron for precise timing/isolated work, heartbeat when work benefits from session context.

His arc: manual 3×3 terminal grid (["Just Talk To It"](https://steipete.me/posts/just-talk-to-it), Oct 2025) → parallel unattended runs (Dec 2025) → "give it cron jobs and a heartbeat so it can be proactive" ([Clawdbot](https://steipete.me/posts/2026/clawdbot), Jan 2026) → the viral post (Jun 2026).

### 9.3 The anatomy both setups share

Every standing loop in both setups has five parts: **(1) a trigger** (cron tick, heartbeat, webhook, GitHub event); **(2) a standing prompt file as intent** (skill body, `HEARTBEAT.md`, `loop.md`, `VISION.md`) so ticks don't re-derive goals; **(3) the model as the per-tick decision-maker** — it decides *whether* and *what*, not a hardcoded branch; **(4) a verifier that can say no** (tests, CI, review findings, live-proof gates) — per the most-cited reply to Steinberger's post: "a loop with nothing to push back is the agent agreeing with itself on repeat"; **(5) an explicit stop/silence condition** (`HEARTBEAT_OK`, "no findings remain", auto-archive-if-nothing-to-report) plus narrowly-privileged deterministic executors for side effects.

### 9.4 Published standing loops you can read/run today

- **[opensanctions/opensanctions issues-agent.yml](https://github.com/opensanctions/opensanctions/blob/main/.github/workflows/issues-agent.yml)** — the strongest production example: twice-daily cron → `tasks.py` scans the ETL warning index and emits a task matrix → claude-code-action per dataset (Sonnet for YAML fixes, Opus for code, per-task max_turns, max-parallel 4), deduped against open autofix PRs; prompt says "Skip ambiguous warnings — defer to human review."
- **[Arize-ai/phoenix weekly deps upgrade](https://github.com/Arize-ai/phoenix/blob/main/.github/workflows/claude-weekly-deps-upgrade.yml)** — deterministic upgrade + 15 verification checks; Claude invoked *only on failure*; the agent is denied git/gh (the workflow commits) and protected paths are restored before commit so it can't edit its own automation.
- **[anthropics/claude-code-action examples](https://github.com/anthropics/claude-code-action/tree/main/examples)** — `issue-triage.yml` (fires on issue open), `ci-failure-auto-fix.yml` (fires on CI failure, with a branch-prefix guard against self-triggering), plus the docs' daily-report cron.
- **[cassioerodrigues/auto-skedway](https://github.com/cassioerodrigues/auto-skedway)** — the clearest "Boris-style" reimplementation: a ~250-line `auto-resolve-issues.sh` on cron — pick 3 oldest open issues → branch → `timeout 30m claude -p --dangerously-skip-permissions` → JSON sentinel check → verify a commit exists → `gh pr create` → comment on the issue; no auto-merge.
- **[cobusgreyling/loop-engineering](https://github.com/cobusgreyling/loop-engineering)** — seven reusable patterns (Daily Triage, PR Babysitter, CI Sweeper, Dependency Sweeper, Changelog Drafter, Post-Merge Cleanup, Issue Triage); [bearded-giant/claude-code-config](https://github.com/bearded-giant/claude-code-config) has a real `/post-merge-sweeper` skill body.
- **Fleets**: [Steve Yegge's Gas Town](https://github.com/steveyegge/gastown) ("Kubernetes for AI coding agents" — Deacon patrol cycles, Witness stuck-detection, Refinery merge queue, Beads ticket queue; ~$100/hr at 12–30 agents) and [paperclipai/paperclip](https://github.com/paperclipai/paperclip) (Matt Van Horn: ticket queue + heartbeats — "Agents wake on a schedule, check work, and act"; "If it can receive a heartbeat, it's hired").
- The HN-canonical backlog pattern in one line: *"while true: if tickets exist → burn down the backlog by one ticket, exit; if not → figure out what feature would make sense to add next, create PRD and ERD, break down into tickets, exit."*
