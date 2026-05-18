# Agentic World: Applying Code-Understanding & Testability Techniques to AI Coding Agents

A companion to `README.md`. The question: now that "agents" are writing
software (Claude Code, Codex, Devin, Aider, Cline, Cursor), how are those
agents themselves being engineered for software quality? Which classical
techniques from the legacy-code / behavioral-analysis world have been
reinvented? What does OpenAI's "software factory" — and Anthropic's, and
Meta's, and DeepMind's — actually look like inside?

---

## Big picture: the mapping

| Classical technique | Agentic counterpart |
|---|---|
| Characterization tests | **RLEF / AlphaCodium / ReVeal** execution-feedback loops |
| Approval testing | **LLM-as-judge** + **snapshot/golden tests** for agent outputs |
| Seams / dependency injection | **Tool registration**, **MCP servers** |
| Repo navigation / Sourcetrail | **Aider's tree-sitter repo map** + **PageRank** |
| Architecture Decision Records | **CLAUDE.md / AGENTS.md / SKILL.md** |
| WELC test harness | **Sandboxes**: E2B, Daytona, Modal, Inspect Sandbox |
| Hotspot prioritisation | **Eval suite design** — which tasks fail most |
| Mutation testing | **Meta's TestGen-LLM / Mutahunter** + property-based eval |
| Code review by senior eng. | **Multi-agent review** + **verifier model** |
| Living Documentation | **Skills**, **plugin docs**, exec'd-in-context notebooks |
| Behavioral code analysis | **SWE-bench / SWE-Lancer / SWE-rebench** mining real GH issues |

The big new term: **"harness engineering"** — building the deterministic
infrastructure around the model (context plumbing, tool routing, recovery,
sandboxes, file editing tools). Since 2025 it's "one of the most frequently
repeated keywords in Anthropic's internal blog posts and in OpenAI's agent
research" ([source](https://www.mindstudio.ai/blog/what-is-agent-harness-architecture-explained)).

---

## 1. Agent harnesses (the new IDE / WELC test harness)

What sits between the model weights and the real world: context
management, tool registry, file I/O wrappers, retry / error recovery,
permission / sandboxing, observability.

**Examples**
- **Claude Code** (Anthropic) — terminal-first agentic CLI; "majority of
  Anthropic's code is now written by Claude Code", 67% PR-merge uplift
  ([VentureBeat](https://venturebeat.com/orchestration/anthropic-says-claude-code-transformed-programming-now-claude-cowork-is)).
  Architecture: reactive ReAct loop, layered policy enforcement, deny-first
  defaults.
- **OpenAI Codex** — multi-concurrent agents with cloud worktrees;
  Codex CLI exposes itself as an MCP server so the Agents SDK can
  orchestrate it ([OpenAI](https://openai.com/index/unrolling-the-codex-agent-loop/), [Harness engineering](https://openai.com/index/harness-engineering/)).
- **Aider** — open source; canonical implementation of a tree-sitter repo
  map + PageRank ranking (more in §3).
- **Cline** — open source IDE agent; monitors linter/compiler errors
  in-the-loop and fixes typos/missing imports before showing the user.
- **Devin** (Cognition AI) — autonomous: plans, executes, iterates over
  thousands of decisions; v2 introduced parallel sessions.
- **Terragon** — cloud orchestrator for *running other agents* (Claude
  Code, Codex, Amp, Gemini) in parallel.

**Pattern catalog**: see [Awesome Harness Engineering](https://github.com/ai-boost/awesome-harness-engineering),
Addy Osmani's [blog](https://addyosmani.com/blog/agent-harness-engineering/),
the [VILA-Lab "Dive into Claude Code"](https://github.com/VILA-Lab/Dive-into-Claude-Code)
paper, and Jonathan Fulton's ["Inside the Agent Harness"](https://medium.com/jonathans-musings/inside-the-agent-harness-how-codex-and-claude-code-actually-work-63593e26c176).

---

## 2. Sandboxes — the WELC "test harness" for code-running agents

Feathers wrote about getting code "into a test harness". Today's agents
need the inverse: a place where *they* can run *arbitrary* code without
trashing your laptop.

| Sandbox | Tech | Cold start | Notes |
|---|---|---|---|
| **[E2B](https://e2b.dev/)** | Firecracker microVMs | ~150 ms | 88% of Fortune 100 signed up |
| **[Daytona](https://daytona.io/)** | Docker | ~90 ms | Persistent sandboxes, auto-stop policies |
| **[Modal](https://modal.com/)** | gVisor (Kata opt-in) | variable | GPU-first, ML workloads |
| **[Sprites.dev](https://sprites.dev/)** | Firecracker | <125 ms | Niche |
| **[Inspect Sandboxing Toolkit](https://www.aisi.gov.uk/blog/the-inspect-sandboxing-toolkit-scalable-and-secure-ai-agent-evaluations)** | Multiple back-ends | n/a | UK AISI's eval-grade sandbox |
| **CodeSandbox**, **Replit**, **StackBlitz** | Browser/server | varies | Older, ergonomic |

Mapping: a sandbox is to an agent what a seam is to legacy code — the
isolation boundary that makes the system testable / runnable safely.

---

## 3. Repo maps — automated hotspot/knowledge-map for the agent

Classical: Sourcetrail, NDepend, CodeScene knowledge maps for humans.

Agentic: the agent itself needs to "understand the codebase" before
writing. The canonical implementation is **Aider's repo map** ([blog post](https://aider.chat/2023/10/22/repomap.html)):

1. **Tree-sitter** parses every file → AST with definitions and references.
2. Build a graph: files = nodes, identifier-references = edges.
3. **PageRank** the graph with personalization toward (a) files in the
   chat, (b) identifiers the user mentioned. Weighted multipliers:
   mentioned id × 10, well-named id × 10, chat files × 50.
4. Binary-search the token budget to pick the top-N symbols that fit.
5. Cache on disk by mtime.

Supports 130+ languages via tree-sitter. This is *exactly* an automated
hotspot/coupling analysis in service of the model's context window.

Other implementations: Sourcegraph Cody's RAG, Cursor's embedding index,
Claude Code's tree-walking grep + read, RepoMapper (an MCP server), Glean
(Meta) as a fact graph.

---

## 4. Eval as the new TDD — SWE-bench family

Where Feathers said "tests first", agent builders say "evals first".

| Benchmark | What it tests |
|---|---|
| **[SWE-bench](https://www.swebench.com/)** | 2 294 real GitHub issues across 12 popular Python repos; agent must produce a patch that passes the hidden test |
| **SWE-bench Verified** | 500-task subset, human-validated as actually solvable; built with OpenAI Preparedness |
| **[SWE-Lancer](https://openai.com/index/swe-lancer/)** | 1 400+ real Upwork freelance tasks, total real-world payout $1 M; mixes coding tasks and *manager* tasks (pick between proposals). Best score so far: Claude 3.5 Sonnet earned ~$400 k |
| **[SWE-rebench](https://swe-rebench.com/)** | 21 000+ tasks, continuously updated and decontaminated against model release dates; designed for RL training of agents |
| **Cybench, GAIA, GDM CTF** | Adjacent — security/agency capability evals |

Insight: SWE-bench mining is *the same forensic-VCS analysis from
Tornhill's work*. Find which GH issues had a definitive resolving commit,
extract them, gate them with the hidden tests that originally validated
the fix. Behavioral analysis turned into training/eval data.

---

## 5. Verifier loops — characterization tests reborn

The most successful technique for raising agent code quality: don't trust
the model — execute the code, capture the result, feed it back.

Notable systems:
- **RLEF** ([Meta, 2024](https://arxiv.org/abs/2410.02089)) — Reinforcement
  Learning with Execution Feedback. Public tests are run after every
  attempt; PPO updates the model with reward = pass/fail. Models trained
  with RLEF outperform baselines while using 10× fewer samples.
- **[AlphaCodium](https://arxiv.org/abs/2401.08500)** (CodiumAI / Qodo) —
  "flow engineering". Iterative loop: generate spec → public tests →
  candidate code → run, fix, expand tests, re-run. +44% accuracy on
  CodeContests vs single-shot.
- **[AlphaCode](https://deepmind.google/blog/competitive-programming-with-alphacode/)** (DeepMind, 2022) — original; massive sampling + test-based filtering.
- **[AlphaEvolve](https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)** (DeepMind, 2024-25) — Gemini-based evolutionary coding agent. The "fitness function" is *any* automated evaluator; AlphaEvolve evolved better matrix-multiplication algorithms and data-center scheduling code.
- **[ReVeal](https://arxiv.org/html/2506.11442v1)** — agent generates its own tests, then uses them as a self-verifier; dense per-turn reward.
- **[Agentic RL for Real-World Code Repair](https://arxiv.org/html/2510.22075)** (2025) — extends RLEF to whole-repo bug fixes.

These are *characterization tests as reward signals*. The "approve the
current behavior" pattern is now "approve only what passes execution and
let RL eliminate the rest."

---

## 6. Automated test generation — the agentic descendant of approval testing

If WELC says "wrap untested code in characterization tests before changing
it", today's tools do that automatically.

- **[Meta TestGen-LLM](https://arxiv.org/pdf/2402.09171)** — used internally
  at Meta. Augments an existing test class with new tests; *only keeps a
  test if it builds, passes reliably, and increases coverage*. At a Meta
  test-a-thon 11.5% of all classes were improved; 73% of recommendations
  shipped to production.
- **[Meta ACH (Automated Compliance Hardening)](https://engineering.fb.com/2025/02/05/security/revolutionizing-software-testing-llm-powered-bug-catchers-meta-ach/)** — mutation-guided. Generate plausible faults; generate
  tests that catch them. This is literally PIT/Stryker × LLM.
- **[Qodo Cover-Agent](https://github.com/qodo-ai/qodo-cover)** — open source. Iterates until a coverage target is hit. First public TestGen-LLM clone.
- **Mutahunter** — open source. Mutation-guided variant. Tests must kill mutants.
- **Diffblue Cover** (commercial, Java) — symbolic + ML test generation; precursor to the LLM wave.
- **Multi-agent variants** — [CANDOR](https://arxiv.org/abs/2506.02943) ("Hallucination to Consensus") uses multiple agents to suppress test hallucinations and beat EvoSuite on mutation score.

These tools translate "approval test the legacy code" from a manual TDD
chore into a CI job an agent runs every night.

---

## 7. Context files — ADRs for agents

A code-base-resident standing instruction set the agent reads on every
session. The same role that ADRs play for human teammates.

| File | Scope |
|---|---|
| **`CLAUDE.md`** | Anthropic-specific session prompt; loaded automatically; walks up the directory tree |
| **`AGENTS.md`** | Cross-tool open standard ([agentsmd.net](https://agents.md/)) for build commands, test commands, conventions, security |
| **`SKILL.md`** | Inside Anthropic Agent Skills folders; YAML frontmatter + markdown |
| **`CONTEXT.md`, `MEMORY.md`, `RULES.md`** | Various conventions across Cursor, OpenCode, Continue, etc. |
| **Cursor `.cursor/rules`** | Cursor's variant |

Best practice (Anthropic's HumanLayer & Augment Code guides): 200-500
words, include exact command strings (`pnpm test:integration`,
`make build-docker`), name conventions, architectural constraints,
security requirements. Treat as living docs; commit to git.

This is **executable Living Documentation** of the Cyrille-Martraire kind,
but specifically for the AI collaborator. References: ["Writing a good
CLAUDE.md" — HumanLayer](https://www.humanlayer.dev/blog/writing-a-good-claude-md);
[Augment Code AGENTS.md guide](https://www.augmentcode.com/guides/how-to-build-agents-md).

---

## 8. Agent Skills / Plugins — modular tools as seams

Anthropic Skills are folders of instructions + scripts + resources Claude
loads *dynamically*. Pre-built for PowerPoint, Excel, Word, PDF; the
[anthropics/skills](https://github.com/anthropics/skills) repo has 17
open-source skills.

Anthropic also shipped a **Skill-Creator plugin** that *evaluates* whether
a skill actually works — i.e. the same eval-driven discipline applied to
the agent's own toolkit.

Analogous in OpenAI land: **Codex MCP server**, **Agents SDK**. The
[Model Context Protocol](https://modelcontextprotocol.io/) is now the
universal tool/seam interface — every editor agent and every IDE-side
tool speaks MCP.

Mapping to classical idea: tools-via-MCP = *object seams* between the
model and the world. Swap an MCP server in tests for a stub MCP server in
production. The Feathers playbook, transposed.

---

## 9. Multi-agent review — code review redux

- **Devin** runs multiple agents in parallel: one writes, others review.
- **Cursor / Claude Code** support sub-agents — "spawn a code-reviewer
  agent over this diff before I commit".
- Microsoft **AutoGen**, **MetaGPT** — explicit role assignment (PM /
  Architect / Engineer / Reviewer). MetaGPT delivers a whole pipeline from
  a one-line requirement.
- **AgentMesh** (2025 paper) — cooperative multi-agent SDLC framework.
- **CodeRabbit, PR-Agent, CodeAnt AI, Greptile, cubic** — purpose-built AI
  PR reviewers; deeper than GitHub Copilot Code Review on architectural
  concerns ([comparison](https://www.cubic.dev/blog/the-3-best-github-copilot-code-review-alternatives-in-2025)).
- **GitHub Copilot Code Review** (GA 2025) — agentic, uses tool-calling to
  explore the repo, combines LLM with ESLint + CodeQL deterministic checks.

Caveat from the research: **LLM-as-judge for code is fragile**. SOTA
models hallucinate "bugs that aren't there" or accept buggy code. Best
practice is to combine judgment with *execution*; treat the LLM judge
output as a hypothesis that an actual test/sandbox must confirm
([Maxim](https://www.getmaxim.ai/articles/measuring-llm-hallucinations-the-metrics-that-actually-matter-for-reliable-ai-apps/)).

---

## 10. Agent observability & eval frameworks

Where SonarQube was for code health, these are for agent health.

- **[Braintrust](https://www.braintrust.dev/)** — release-gate evals, prompt
  versioning, trace inspection.
- **[Langfuse](https://langfuse.com/)** — OSS, self-hosted, prompt
  versioning + traces.
- **[LangSmith](https://smith.langchain.com/)** — LangChain stack.
- **[Arize Phoenix](https://phoenix.arize.com/)** — OSS observability.
- **[OpenAI Evals](https://github.com/openai/evals)** — the original eval
  framework.
- **[Inspect AI](https://inspect.aisi.org.uk/)** (UK AI Security Institute)
  — eval-grade framework for agentic capability tests; Inspect Evals
  collection includes SWE-Bench, GAIA, Cybench, GDM CTF.
- **[Galileo, Latitude, Confident AI / DeepEval, Comet Opik, MLflow](https://mlflow.org/top-5-agent-observability-tools/)** — various commercial / OSS angles.

Convergence: every serious agent stack now ships an eval suite *first*
and a model *second*. This is Tornhill's "put numbers on gut feelings"
applied to the agents themselves.

---

## 11. What OpenAI's "software factory" actually looks like

Synthesised from OpenAI's own posts ([Unrolling the Codex agent
loop](https://openai.com/index/unrolling-the-codex-agent-loop/),
[Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/),
[Harness engineering](https://openai.com/index/harness-engineering/)):

1. **Codex CLI** — terminal agent with shell + file tools.
2. **Codex Cloud** — cloud worktrees, parallel agents per project, with
   sandboxed execution.
3. **App Server** — the orchestration layer that lets multiple agents
   collaborate, share state, run tasks in parallel.
4. **MCP everywhere** — Codex CLI exposed *as* an MCP server so the
   Agents SDK can drive it; every external service plugged in via MCP.
5. **Agents SDK** — deterministic, reviewable workflows that scale from a
   single agent to a delivery pipeline.
6. **Evals** — SWE-bench Verified, SWE-Lancer, internal evals; PRs are
   gated by eval deltas.
7. **OpenAI Preparedness** — the safety team that co-developed SWE-bench
   Verified; basically the "QA team" for the factory.

Internally Anthropic uses Claude Code itself ("dogfooding"); OpenAI uses
Codex; Cognition uses Devin to maintain Devin. The recursion is the
giveaway: when you trust your factory to build the factory, you've
hardened the harness enough to ship.

---

## 12. Recommended reading list (agentic)

Engineering blog posts:
- Anthropic — [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Anthropic — [Writing effective tools for AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Anthropic — [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Anthropic — [Building Effective AI Agents: Architecture Patterns](https://resources.anthropic.com/hubfs/Building%20Effective%20AI%20Agents-%20Architecture%20Patterns%20and%20Implementation%20Frameworks.pdf)
- Anthropic — [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- OpenAI — [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- OpenAI — [Harness engineering](https://openai.com/index/harness-engineering/)
- Aider — [Building a better repository map with tree-sitter](https://aider.chat/2023/10/22/repomap.html)
- Meta Engineering — [Revolutionizing software testing: LLM-powered bug catchers](https://engineering.fb.com/2025/02/05/security/revolutionizing-software-testing-llm-powered-bug-catchers-meta-ach/)

Papers:
- [SWE-bench (Jimenez et al., ICLR 2024)](https://arxiv.org/abs/2310.06770)
- [SWE-Lancer (OpenAI, 2025)](https://arxiv.org/abs/2502.12115)
- [SWE-rebench (Nebius, 2025)](https://arxiv.org/abs/2505.20411)
- [RLEF (Meta, 2024)](https://arxiv.org/abs/2410.02089)
- [AlphaCodium (CodiumAI, 2024)](https://arxiv.org/abs/2401.08500)
- [Mutation-Guided LLM-based Test Generation at Meta (FSE 2025)](https://arxiv.org/abs/2501.12862)
- [Automated Unit Test Improvement at Meta — TestGen-LLM](https://arxiv.org/pdf/2402.09171)
- [ReVeal: Self-Evolving Code Agents](https://arxiv.org/html/2506.11442v1)
- [Agentic RL for Real-World Code Repair](https://arxiv.org/html/2510.22075)
- [CANDOR — Hallucination to Consensus (multi-agent JUnit)](https://arxiv.org/abs/2506.02943)

Books / longer:
- *Building Effective AI Agents* (Anthropic, free PDF).
- 12 Factor Agents — Dex Horthy's [principles](https://paddo.dev/blog/12-factor-agents/), inspired by 12 Factor App.

Industry pattern catalogs:
- [Awesome Harness Engineering](https://github.com/ai-boost/awesome-harness-engineering)
- [Dive into Claude Code — VILA-Lab](https://github.com/VILA-Lab/Dive-into-Claude-Code) (systematic analysis of Claude Code's design space)
- [12 Agentic Harness Patterns from Claude Code](https://generativeprogrammer.com/p/12-agentic-harness-patterns-from)

---

## 13. The honest summary

The agentic world *re-derived* most of the classical legacy-code playbook
under new names:

- "Characterization test" → "execution feedback" / "verifier loop".
- "Test harness" → "sandbox".
- "Seam" → "tool / MCP server".
- "Hotspot prioritisation" → "eval task selection".
- "ADR" → "CLAUDE.md / AGENTS.md".
- "Architecture-as-tests" → "deterministic harness scaffolding".
- "Mutation testing" → "mutation-guided LLM test generation" (Meta ACH).
- "TDD" → "evals-first development".

The one genuinely new piece is **context engineering**: deciding what
goes into the model's limited window — what to retrieve, what to compress,
when to spawn a sub-agent vs. continue in the parent context. There's no
clean classical analogue because human developers don't have a "context
window".

Net: the field has *converged* on the same insight Feathers and Tornhill
arrived at twenty years ago — **execution beats opinion, behavior beats
spec, evidence beats intuition** — but with sandboxed loops and RL where
the practitioners used to have a copy of WELC and a coffee.
