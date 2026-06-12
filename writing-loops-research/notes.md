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

### Agent 2 — Ralph Wiggum deep-dive (done)

Primary sources (Huntley):
- ghuntley.com/ralph/ (Jul 2025): "Ralph is a technique. In its purest form, Ralph
  is a Bash loop." `while :; do cat PROMPT.md | npx --yes @sourcegraph/amp ; done`
  "deterministically bad in a nondeterministic world"; "tune it like a guitar";
  $50k contract delivered for $297 in API costs; "Engineers are still needed."
- ghuntley.com/loop/ (~Jan 2026): "everything is a ralph loop"; anti-multi-agent
  ("microservices... non-deterministic — a red hot mess"); "sit on the loop";
  "watch the loop as that is where your personal development... will come from."
- ghuntley.com/cursed/ (~Sep 2025): 3-month loop produced "cursed", a Gen-Z slang
  programming language (slay=fn, sus=var) with LLVM backend. PC Gamer covered it.
- ghuntley.com/pressure/ (backpressure: tests/typecheckers reject bad generations);
  ghuntley.com/allocations/ (context allocation theory; MCP wastes context).
- ghuntley/how-to-ralph-wiggum repo (Dec 2025, with Clayton Farr's ralph-playbook):
  "3 Phases, 2 Prompts, 1 Loop" — specs/*.md → PLANNING loop (gap analysis →
  IMPLEMENTATION_PLAN.md) → BUILDING loop (one task/iteration, commit, exit).
  loop.sh: `cat $PROMPT_FILE | claude -p --dangerously-skip-permissions
  --output-format=stream-json --model opus --verbose` in while true; max-iterations;
  git push per iteration. PROMPT_build.md: "Study specs/* with up to 500 parallel
  Sonnet subagents", "only 1 subagent for build/tests" (backpressure), escalating-9s
  guardrails. Context: ~176K usable, 40-60% utilization "smart zone".

Official adoption & controversy:
- Anthropic official plugin: anthropics/claude-code plugins/ralph-wiggum —
  /ralph-loop "task" --completion-promise "DONE" --max-iterations 50; implemented
  via Stop hook re-feeding prompt IN ONE SESSION (context accumulates!) — issue
  #125 "deviates from original Ralph behavior (context should be fresh each
  iteration)"; Huntley + Dex Horthy video "why Claude Code's implementation isn't
  it" (youtube O2bBWDoxO4s). Curio: claude-code#23084 model-welfare complaint.

History/amplification:
- humanlayer.dev/blog/brief-history-of-ralph (Dex Horthy): "naive persistence";
  "the LLM isn't protected from its own mess; it is forced to confront it."
- Viral wave Dec 2025–Jan 2026: Matt Pocock (aihero.dev guides), Ryan Carson
  (690k+ views); Huntley pushed back on both simplifications.
- VentureBeat "How Ralph Wiggum went from The Simpsons to the biggest name in AI".

Implementations:
- snarktank/ralph (Ryan Carson): PRD-driven, prd.json + progress.txt + git as
  memory, "Each iteration is a fresh instance with clean context", MAX_ITERATIONS=10.
- snwfdhmp/awesome-ralph list: mikeyobrien/ralph-orchestrator (Rust, 7 backends,
  "Hat" personas), vercel-labs/ralph-loop-agent, iannuttall/ralph,
  frankbria/ralph-claude-code (circuit breaker), agrimsingh/ralph-wiggum-cursor
  (context rotation @80k tokens), Block Goose ralph-loop tutorial, r/RalphCoding.

Critiques/reports:
- HN id=44565028 (original), id=45005434 (repomirror "6 repos overnight"),
  id=46672413 "Ralph Wiggum Doesn't Work" (Pedro Nauck: cascade failures — "Jenga
  tower of broken code by Task 10"; needs isolation/verification/persistence).
- codecentric report: phase isolation + explicit verification phase; "the better
  your spec, the better the output". Failure modes: "overcooking" vs "undercooking".
- Steve Kinney "Entering the Mind of Ralph Wiggum" (Mar 2026): "your files and git
  history are a better memory layer than the LLM's context window"; 4 components
  (bash=dumbest part, PROMPT.md=brain, filesystem=memory, backpressure=verification);
  ~$10/hr API burn.
- Alibaba Cloud: "From ReAct to Ralph Loop: A Continuous Iteration Paradigm".
- Amp added amp.experimental.autoHandoff (handoff at 90% context).

### Agent 4 — Prompt structures & harness designs (done; prompts fetched verbatim)

Anthropic primary sources:
- "Effective harnesses for long-running agents" (Nov 2025): initializer agent +
  coding agent; init.sh, claude-progress.txt, 200-item feature_list.json (all
  passes:false), git commits; smoke test at session start.
- anthropics/claude-quickstarts/autonomous-coding — actual prompts verbatim:
  coding_prompt.md opens "This is a FRESH context window - you have no memory of
  previous sessions"; 10-step checklist (GET YOUR BEARINGS → init.sh → regression
  check → CHOOSE ONE FEATURE → implement → browser-verify ("curl is insufficient")
  → only modify 'passes' field → commit → update progress → end cleanly).
  initializer: "IT IS CATASTROPHIC TO REMOVE OR EDIT FEATURES IN FUTURE SESSIONS."
- "Harness design for long-running apps" (Mar 2026): planning/generation/evaluation
  split into agents; with Opus 4.5 context resets were DROPPED in favor of one
  continuous session + Agent SDK auto-compaction. "Every component in a harness
  encodes an assumption about what the model can't do on its own... can go stale."
- anthropics/cwc-long-running-agents (Code with Claude 2026): three primitives —
  default-FAIL contract (PreToolUse hook denies writes to test-results.json
  without evidence read first; "the harness makes 'done' structural"),
  fresh-context evaluator subagent (PASS/NEEDS_WORK; "the builder shouldn't grade
  its own work"), agent-maintained handoff (PROGRESS.md Done/In progress/Next/
  Notes). Verbatim outer loop:
    while grep -q '"passes": false' test-results.json; do
      claude -p "Read PROGRESS.md and build the next unfinished feature..."
      VERDICT=$(claude --agent evaluator -p "Review the most recent commit...")
      [ "$(head -1 <<<"$VERDICT")" = "PASS" ] || echo "$VERDICT" > NEXT_FINDINGS.md
    done
  Plus kill-switch.sh (AGENT_STOP file), steer.sh (STEER.md), built-in /goal.
- Nicholas Carlini C compiler (Feb 2026): 16 parallel agents, ~2000 fresh sessions,
  $20K, 100K-line Rust C compiler builds Linux 6.9; lock files in current_tasks/
  synced via git; massive test-suite backpressure.
- Agent SDK loop: gather context → take action → verify work → repeat.
- Context engineering post: compaction vs structured note-taking vs sub-agents;
  Pokémon memory failure (31 files, transcript-not-state, stuck in second town).
OpenAI:
- "Unrolling the Codex agent loop"; long-horizon guidance = durable project memory
  in markdown; /responses/compact endpoint; /goal productized outer loop.
  Continuation philosophy: compaction (one long session) vs Ralph's stateless restart.
Others:
- obra/superpowers: subagent-per-task + two-stage review; "Continuous execution:
  Do not pause to check in... only reasons to stop are BLOCKED, ambiguity, or all
  tasks complete."
- 12-factor/Horthy "dumb zone": past ~40% context utilization quality degrades —
  empirical basis for fresh-context iterations.
- frankbria/ralph-claude-code: dual-condition exit (completion indicators AND
  EXIT_SIGNAL:true in RALPH_STATUS block) + circuit breakers.
- HN: Opus 4.5 ran 4h49m via stop hooks; Matt Van Horn loops PRs across ~30 repos
  overnight; Cherny: "always give Claude a way to verify its work... you can't
  manually review a 26-hour thread—the system must verify itself."

Synthesis (consensus patterns) captured for README:
- Prompt structure: fresh-context identity → mandatory orientation ritual →
  regression check → ONE task scope clamp → search-before-assume → verification
  with anti-shortcut rules → end-of-iteration ritual → guardrails.
- Persistent state: plan/task file (append-only goals + pass flags), curated
  progress/handoff file, lean operational AGENTS.md, git, evidence artifacts.
- Stop conditions menu: work-queue-empty (data not vibes), completion promise,
  independent evaluator, hard budgets, stall detection, human escape hatches.

## Verification pass

Cross-checked across independent agents:
- Cherny quote: agents 1 & 5 both found it; venue = Anthropic dev conf SF
  (~Jun 5-6, 2026) per Firecrawl/productmarketfit/OfficeChai; one source said
  "Acquired podcast" — treated as misattribution (likely conflation with his
  Dec 2025 Peterman Pod interview). Confidence: quote text high, venue medium.
- Steinberger tweet: 2 agents, consistent text & status ID. High confidence.
- Osmani "Loop Engineering" ~Jun 9 2026: 3 agents converge. High confidence.
- Ralph loop one-liner & playbook details: repo cloned, verbatim. High confidence.
- sketch.dev 9-line loop, Anthropic quickstart prompts, cwc loop, 12-factor
  pseudocode: fetched from raw.githubusercontent / mirrors. High confidence.
- Mitchell Hashimoto "harness engineering" coinage (Feb 2026): secondary sources
  only. Medium confidence — flagged in README.
- View counts (700K/2.2M/6.5M): inconsistent across sources; report as approximate.

## Wrap-up

Wrote README.md synthesizing all five angles with citations. Final commit includes
only notes.md + README.md (no fetched code copies, per repo instructions).

## Follow-up (same day): "the loop Boris/Steinberger meant" — standing/trigger-driven loops

User clarified: not the Ralph/goal-style iterate-until-done loop, but standing
loops that ORIGINATE work (cron/event → decide → prompt agent). Launched 3 agents:
Cherny's actual loops, Steinberger/OpenClaw loops, trigger-driven examples in wild.

### Agent C — Trigger-driven loop implementations (done)

Official/vendor:
- Claude Code **Routines** (code.claude.com/docs/en/web-scheduled-tasks): saved
  prompt + repos + connectors; triggers = cron (min 1h), per-routine HTTP API
  endpoint (POST /v1/claude_code/routines/<id>/fire), GitHub events with filters.
  Safety: claude/-prefixed branches only, daily run caps. Also /loop (in-session
  recurring, docs/en/scheduled-tasks) and Desktop local scheduled tasks.
- anthropics/claude-code-action examples: issue-triage.yml (on issues:opened),
  ci-failure-auto-fix.yml (on workflow_run completed + failure, with branch-prefix
  guard against self-trigger), daily-report cron example in docs.
- Headless docs: claude -p --bare + --allowedTools + --max-turns = the cron surface.
- OpenAI Codex Automations: cron syntax, results to inbox, auto-archives if
  nothing to report. Devin Scheduled Sessions + Playbooks (cron, Slack/GitHub/
  Linear/webhook triggers, invocation limits).
Production OSS standing loops:
- **opensanctions/opensanctions issues-agent.yml** — strongest real example:
  2x-daily cron → tasks.py scans ETL warning index → JSON task matrix → claude-
  code-action per dataset (Sonnet for YAML fixes, Opus for code), max-parallel 4,
  dedupe vs open autofix PRs and closed identical branches, "skip ambiguous
  warnings — defer to human review".
- JetBrains/ideavim updateChangelogClaude.yml (daily 5am cron, Claude opens PR).
- Arize-ai/phoenix claude-weekly-deps-upgrade.yml: deterministic upgrade + 15
  verify checks logged to .scratch/verify/, Claude invoked ONLY on failure; agent
  denied git/gh; protected paths restored before commit.
- speed47/spectre-meltdown-checker vuln-watch.yml: daily; Claude step skipped if
  nothing new; report-only output.
- sandgraal/compass agent-nightly-triage.yml: memory file + single rolling issue
  ("🌙 Nightly bug triage") instead of issue spam; repo-variable kill switch.
- wpbluiss/conduit-nextjs auto-review-merge.yml: PR events + 30-min cron safety
  net; safety floor (never auto-merge destructive/secret-exposing); gh pr merge.
Fleets/daemons:
- **Steve Yegge Gas Town** (steveyegge/gastown): "Kubernetes for AI coding agents",
  20-30 Claude instances; Deacon patrol cycles, Witness stuck-detection, Refinery
  merge queue; work from Beads issue tracker; ~$100/hr.
- **paperclipai/paperclip** (Matt Van Horn): ticket queue + heartbeats, "Agents
  wake on a schedule, check work, and act"; "If it can receive a heartbeat, it's
  hired"; ACP plugin binds coding agents to chat threads.
- **OpenClaw**: cron jobs (~/.openclaw/cron/, isolated session per job) +
  heartbeat (default 30 min, HEARTBEAT.md checklist); cron-vs-heartbeat doc;
  template repo converts prompt jobs to deterministic scripts over time.
- Small cron+claude -p: jshchnz/claude-code-scheduler (launchd/crontab plugin),
  mvanhorn/agentmail-to-claude-code (email → session), hum-growth/hum,
  bakernyc/autonomous-trading-agent (limits in CLAUDE.md), brad-pierce/dreamcatcher,
  AnandChowdhary/continuous-claude (PR-native bridge case).
- HN canonical pattern (item 46632445): "while true: if tickets exist -> burn down
  backlog by one ticket, exit; if not -> figure out what feature would make sense,
  create PRD/ERD, break into tickets, exit."
Cross-cutting patterns: deterministic work-origination + agent-as-fixer; dedupe/
idempotency guards; hard limits (turns/parallel/budget/tool allowlists/branch
prefixes); verification before merge; report-don't-act defaults.

### Agent S — Steinberger's actual loops (done; code fetched verbatim from GitHub)

The viral post (Jun 7-8, 2026, ~6.5M views): "monthly reminder" framing — doctrine
he'd demonstrated since May 2026. His thread reply to "how?": point at VISION.md
("so each loop tick does not re-derive intent from scratch"). Circulating example:
"/loop babysit all my PRs. Auto-fix build issues, and when comments come in, use a
worktree agent to fix them." Top reply (@mosyaseen): "the other half is putting
something in the loop that can say no: a test, a type check, a real error. a loop
with nothing to push back is the agent agreeing with itself on repeat."
Distillation: "a loop is cron plus a decision-maker."

His actual loops:
- autoreview skill (openclaw/agent-skills skills/autoreview/SKILL.md; symlinked in
  steipete/agent-scripts, 4.8k stars): "runs codex /review in a loop until there's
  no booboos anymore" (tweet 2054850632067019173, May 14 2026). Findings advisory;
  verify independently; loop ends when no accepted findings remain.
- maintainer-orchestrator skill: control-plane agent classifying queue items
  (autonomous/owner-decision/ignore), delegates to workers, monitors every 5 min,
  worker contract: regression tests + "execute autoreview until findings are
  resolved" + live-proof gate; release only at zero open issues + green CI.
- ClawSweeper (openclaw/clawsweeper): weekly sweep of all 7k+ issues/PRs; four
  lanes Review (proposes, never applies) / Apply (15-min wake, high-confidence
  unchanged proposals only) / Repair (@clawsweeper autofix → bounded AI fix loop;
  deterministic executor owns every GitHub mutation) / Commit Review. "AI runs
  without GitHub write tokens." Adaptive cadence hot=hourly, <30d=daily, else weekly.
- OpenClaw architecture: 6 mechanisms — Cron, Heartbeat, Background tasks, Task
  Flow, Standing Orders, Hooks. Heartbeat = main-session turn every 30m reading
  HEARTBEAT.md; reply HEARTBEAT_OK suppressed. Cron: SQLite-persisted, at/every/
  cron schedules, isolated cron:<jobId> sessions, announce/webhook delivery,
  maxConcurrentRuns 8 + retry backoff. Standing Orders doc = the tweet encoded:
  "define programs with clear scope, triggers, and escalation rules."
- Arc: "Just Talk To It" (Oct 2025, manual 3x3 grid) → "Shipping at Inference-
  Speed" (Dec 2025, parallel unattended) → "Clawdbot" (Jan 2, 2026: "give it cron
  jobs and a heartbeat so it can be proactive") → OpenAI (Feb 2026) → autoreview/
  crabbox (May) → viral post (Jun). Lex Fridman #491 (Feb 12, 2026).
- Synthesis: every loop = trigger + standing prompt file (intent) + model as
  per-tick decision-maker + verifier that can say no + explicit stop/silence
  condition, with deterministic narrowly-privileged executors for side effects.
