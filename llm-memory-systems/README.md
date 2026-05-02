# LLM / Agentic Memory Systems — A Conceptual Survey

A deep-dive into how the LLM-agent community currently *measures* long-term memory and the **five fundamentally different conceptual approaches** that show up at the top of those leaderboards.

Date of survey: May 2026.

---

## Part 1 — The Benchmarks

A memory system can only be "best" relative to a measurement. Four benchmarks dominate published comparisons; one is a model-not-system leaderboard included for context.

### 1.1 LongMemEval (ICLR 2025)

**Paper:** Wu et al., *LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory*, arXiv 2410.10813

**What it tests.** Five "core memory abilities":

| Ability | What is measured |
|---|---|
| Information extraction | Recall of an explicit fact dropped earlier in a chat |
| Multi-session reasoning | Synthesizing facts that span ≥2 sessions |
| Temporal reasoning | Reasoning about explicit times and timestamp metadata |
| Knowledge updates | Tracking facts that *change* over time (overwrites) |
| Abstention | Refusing to answer when info is genuinely absent |

These are surfaced through six question types: `single-session-user`, `single-session-assistant`, `single-session-preference`, `temporal-reasoning`, `knowledge-update`, `multi-session`. Abstention questions are flagged with an `_abs` suffix (~30 of the 500).

**Dataset.** 500 manually-curated questions embedded in synthesized user/assistant chat histories. Three sizes:
* `LongMemEval_S` — ~115 K tokens / question
* `LongMemEval_M` — ~500 sessions, ~1.5 M tokens / question
* `LongMemEval_oracle` — gold evidence only, an upper bound on the reader

**Framework.** The paper imposes a 3-stage decomposition that subsequent papers all reuse: **Indexing → Retrieval → Reading**. This is what makes LongMemEval the *de facto* unit of comparison — a memory system can be benchmarked at the retrieval stage independently of reader quality.

**Metrics.**
* Answer quality: GPT-4o LLM-as-judge with >97 % agreement against human experts.
* Retrieval (when traces are exposed): `Recall@k`, `NDCG@k`.

**Headline finding.** Commercial assistants drop ~30 % in accuracy when moving from offline full-context reading to online interaction. GPT-4o falls from 91.8 % → 57.7 % on short (3-6 session) histories. Long-context windows alone do not solve memory.

### 1.2 LoCoMo (ACL 2024, Snap Research)

**Paper:** Maharana et al., *Evaluating Very Long-Term Conversational Memory of LLM Agents*, arXiv 2402.17753

**What it tests.** Long, persona-grounded conversations between two agents.

**Dataset.** Each conversation is ~600 turns / ~16 K tokens / up to 32 sessions. Built by a machine-human pipeline: LLM agents are anchored to personas + temporal event graphs, can share images, and human annotators verify long-range consistency.

**Tasks & question categories.**
1. **Question Answering** — five sub-categories: single-hop, multi-hop, temporal, open-domain, adversarial.
2. **Event Summarization** — produce a daily/weekly summary that matches the underlying event graph.
3. **Multi-modal Dialogue Generation** — continuation given prior turns and shared images.

**Metrics.** F1 (extractive), BLEU/ROUGE-style overlap for summarization, GPT-4-as-judge for end-to-end accuracy.

**Why it matters.** LoCoMo is the most-cited test bed because it stresses *aggregation across sessions* and *temporal grounding* together — a system that just stores semantically similar snippets fails the multi-hop and temporal questions.

### 1.3 MemoryAgentBench (ICLR 2026)

**Paper:** Hu et al., *Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions*, arXiv 2507.05257

**What it tests.** Four core competencies, chosen specifically to differentiate memory *agents* (not just stored snippets):

1. Accurate retrieval
2. Test-time learning (acquiring new skills from session history)
3. Long-range understanding
4. **Selective forgetting** (distinguishes from append-only systems)

**Dataset.** Existing long-context datasets are sliced into chunks and fed *incrementally*, simulating real agent interaction. Two new sub-datasets: `EventQA` (retrieval) and `FactConsolidation` (conflict resolution — testing whether memory can resolve contradictions over time).

**Headline finding.** No current method (RAG, MemGPT-style external memory, tool-using agents) scores well on all four competencies at once.

### 1.4 Letta Leaderboard (May 2025)

**Source:** docs.letta.com/leaderboard

This is a *model* leaderboard, not a system one — it benchmarks how well different LLMs handle agentic memory ops with the *same* (Letta) framework underneath. Useful as a confound-controller: it tells you how much performance is the LLM vs the memory architecture.

**Two memory tiers measured:** core memory (in-context) and archival memory (external store), each on read / write / update primitives.

### 1.5 DMR (Deep Memory Retrieval)

The earlier benchmark used in the original MemGPT paper and reused by Zep. Mostly superseded by LongMemEval but still cited (Zep 94.8 % vs MemGPT 93.4 %).

### 1.6 Reported scores at a glance (LongMemEval-S, GPT-4o reader)

> Caveat: vendors run their own configurations; numbers are not directly comparable unless on a shared harness.

| System | LongMemEval-S | LoCoMo (LLM-judge) |
|---|---|---|
| Long-context baseline (full read) | 91.8 % (offline) / 57.7 % (online) | — |
| OpenAI Memory | 52.9 % | 52.9 % |
| Mem0 | 66.9 % | 66.9 % |
| Zep / Graphiti | 71.2–75.1 % | — |
| Memobase | 75.8 % | — |
| Supermemory | 85.4 % | — |
| Hindsight | 89.6 % | — |
| Maximem Synap | 90.2 % | — |
| ByteRover 2.0 | 92.2 % | — |

---

## Part 2 — The Top 5 *Conceptual* Approaches

Below, "top 5" means the five conceptually distinct architectures that dominate the leaderboards. Within each bucket, multiple products exist (e.g., Mem0 and OpenAI Memory both fall in C2); they differ in execution, not in idea.

The five buckets answer the same question — *where do memories live and how do you retrieve them?* — in fundamentally different ways.

### Concept 1 — OS-Paged Hierarchical Memory

**Representative systems:** MemGPT, Letta.
**Reference paper:** Packer et al., *MemGPT: Towards LLMs as Operating Systems*, arXiv 2310.08560.

**Core idea.** Treat the context window like RAM and an external store like disk. The LLM is the CPU; a virtual-memory manager pages information in and out. The agent itself is a *first-class participant* — it calls tools to read/write/move memory.

**Architecture.**
* **Primary context (RAM)** — three partitions:
  * Static system prompt
  * Dynamic *core memory* (small, writeable scratchpad, edited only by tool calls)
  * FIFO *message buffer* (gets evicted to recall storage)
* **External context (disk)**:
  * **Recall storage** — full chat history, queryable
  * **Archival storage** — vector DB for arbitrary long-term notes / data sources

**Retrieval primitive.** The agent decides. It issues `archival_memory_search(query)` or `core_memory_replace(...)` as ordinary tool calls.

**Strengths.** Generality (any data source, anywhere in the hierarchy). The agent's metacognition does the work — no fixed extraction pipeline.

**Weaknesses.** Failure cascades from poor LLM judgement (the agent must remember *to* search). High token cost from tool-call traces. No structured time model.

**Why it's its own concept.** The defining choice is *agent-as-pager*: memory ops are externalized as tools the LLM invokes. Every other category in this list defines memory ops as *implicit* pipelines that run on every turn.

### Concept 2 — Extractive Vector-Based Fact Memory

**Representative systems:** Mem0 (vanilla mode), OpenAI Memory, MemoryBank, Generative Agents (Park et al. 2023).
**Reference paper:** Chhikara et al., *Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory*, arXiv 2504.19413.

**Core idea.** Don't store the conversation; store *atomic facts* extracted from it. Then retrieve facts via dense semantic similarity, the way classic RAG retrieves chunks.

**Pipeline (Mem0 as canonical example).**
1. **Extraction.** An LLM reads each turn and emits atomic memories ("Alice is vegan", "Alice's birthday is March 4").
2. **Consolidation.** New memories are compared against existing ones — duplicates merged, contradictions overwritten.
3. **Storage.** Each memory is embedded and stored in a vector DB.
4. **Retrieval.** At query time, top-K nearest memories by cosine similarity are pulled into context.

**Variations within the bucket:**
* **Generative Agents** rank with `score = recency × relevance × importance` (importance scored by an LLM at write-time). Adds *reflections* — periodic LLM passes that summarize many low-level facts into higher-level abstractions.
* **MemoryBank** adds an Ebbinghaus-style forgetting curve: each memory has a decaying retention strength.
* **OpenAI Memory** (ChatGPT) is the same shape with a thinner extraction pass.

**Performance.** Mem0 reports a 26 % relative improvement over OpenAI Memory on LoCoMo (66.9 % vs 52.9 % LLM-judge), with a 91 % p95 latency reduction vs full-context baselines.

**Strengths.** Cheap, fast, easy to implement, plugs into any vector DB. Latency is dominated by an embedding lookup. Production-ready.

**Weaknesses.** Lossy extraction (whatever the extractor missed is gone). No relational reasoning — "Alice is vegan" and "Bob is married to Alice" are unrelated rows. Temporal reasoning is shallow (timestamps as metadata, not a model). Conflict handling is brittle.

**Why it's its own concept.** The defining choice is *fact-as-row*: memory is a flat collection of independent embedded strings. The next two categories explicitly reject this flatness.

### Concept 3 — Temporal Knowledge Graph Memory

**Representative systems:** Zep / Graphiti, Mem0-Graph mode, Cognee, LightRAG, GraphRAG-style adaptations.
**Reference paper:** Rasmussen et al., *Zep: A Temporal Knowledge Graph Architecture for Agent Memory*, arXiv 2501.13956.

**Core idea.** Memory is a graph of entities and relations, and *every edge has a validity interval*. You retrieve by traversing the graph, not by similarity to a single point.

**Architecture (Graphiti / Zep as canonical).**
* **Nodes.** Entities (people, places, events) with embeddings, types, and a creation timestamp.
* **Edges.** Triplets `(source, relation, target)` carrying a **bi-temporal** stamp: when the fact *occurred* (`t_valid`) and when it was *ingested* (`t_invalid` is set later if the fact is contradicted, never deleted).
* **Communities.** Periodic clustering produces summary nodes — Graphiti's analog of Generative Agents' reflections.
* **Hybrid retrieval.** Queries combine: (a) BM25 lexical match on edge text, (b) cosine similarity on node/edge embeddings, (c) graph traversal from anchor entities. Notably, retrieval involves *no LLM calls* — pure index ops.

**Variations within the bucket:**
* **Mem0-Graph** uses a graph DB (FalkorDB) on top of the same extraction pipeline as Mem0-vanilla. Reports ~2 pp improvement over flat vector mode on LoCoMo.
* **Cognee** is graph + vector hybrid with 14 retrieval modes including chain-of-thought graph traversal; scored well on multi-hop HotPotQA.
* **HippoRAG** (covered separately under Concept 5 — it's a graph but the *retrieval* algorithm is the conceptual signature).

**Performance.** Zep reports 94.8 % on DMR (vs MemGPT 93.4 %) and up to 18.5 % accuracy improvement on LongMemEval with 90 % lower latency than full-context baselines.

**Strengths.** Multi-hop reasoning ("which of Alice's friends are vegan?"). Native time-travel queries ("what did Alice believe in March?"). No information *deleted* — invalidated facts are queryable for provenance.

**Weaknesses.** Extraction cost is higher (must produce typed entities + relations). Schema drift over time. Querying requires more sophisticated patterns than a single similarity search.

**Why it's its own concept.** The defining choice is *relation-as-first-class-citizen with time*. The flat vector store stores points; this stores edges between them, and every edge knows when it was true.

### Concept 4 — Self-Organizing Agentic Memory (Zettelkasten-style)

**Representative systems:** A-MEM, Reflective Memory Management (RMM), H-MEM.
**Reference paper:** Xu et al., *A-MEM: Agentic Memory for LLM Agents*, NeurIPS 2025, arXiv 2502.12110.

**Core idea.** Memories are *notes* that link to other notes, and the network *evolves*. Inspired by Niklas Luhmann's Zettelkasten: each note carries enough metadata to be discoverable, and adding a new note can rewrite the contextual descriptions of older ones.

**Architecture (A-MEM).**
* **Note structure.** Each new memory becomes a note containing:
  * Verbatim content
  * LLM-generated *context* (what kind of memory this is)
  * Keywords, tags
  * Embedding
  * Outgoing links to related historical notes
* **Linking.** When a note is added, the system retrieves semantically similar historical notes and asks the LLM to decide which links to draw.
* **Evolution.** The arrival of a new note can trigger updates to the *context fields* of older notes — i.e., stored memories' interpretations change as more context arrives.
* **Retrieval.** Standard similarity search on note embeddings, then expansion along links.

**Variations.**
* **RMM (Reflective Memory Management)** distinguishes *prospective reflection* (summarizing on the fly across granularities) from *retrospective reflection* (refining retrieval based on what the reader actually cited). It's a feedback loop, not a one-shot extraction.
* **H-MEM** adds a fixed multi-level hierarchy on top.

**Strengths.** Schema-free — no entity/relation taxonomy needed. Reinterprets old memories as new context arrives, partially addressing the "knowledge update" axis without temporal bookkeeping. Handles ambiguous/abstract content better than rigid graphs.

**Weaknesses.** LLM-heavy at write time (every new note triggers context generation, link decisions, and possible rewrites). Harder to reason about cost and latency. Less benchmark coverage than C2/C3 systems — A-MEM's empirical wins are reported on six foundation models against SOTA baselines but it's not yet on the LongMemEval public leaderboard.

**Why it's its own concept.** The defining choice is *the memory store itself reasons*. C2 stores facts and forgets; C3 stores facts and time-stamps; C4 stores facts that *interpret each other*. The graph is implicit and emergent rather than schema-defined.

### Concept 5 — Biologically-Inspired Episodic / Hippocampal Memory

**Representative systems:** HippoRAG, EM-LLM.
**Reference papers:**
* Gutierrez et al., *HippoRAG: Neurobiologically Inspired Long-Term Memory for Large Language Models*, NeurIPS 2024, arXiv 2405.14831.
* Fountas et al., *Human-inspired Episodic Memory for Infinite Context LLMs*, ICLR 2025, arXiv 2407.09450.

**Core idea.** Borrow the *retrieval algorithm* from human neuroscience rather than from databases. Two distinct strands:

**5a — HippoRAG (hippocampal indexing theory).**
* The corpus is converted offline into a schema-less knowledge graph by an LLM (the "neocortex" pass).
* At query time, the LLM extracts query concepts; these are mapped to graph nodes; **Personalized PageRank** is run from those nodes as seeds.
* The PageRank stationary distribution is treated as a "memory activation pattern" — passages most central to the activated subgraph are retrieved.
* Outperforms iterative-retrieval baselines like IRCoT on multi-hop QA by up to 20 % while being 10–30× cheaper and 6–13× faster.

**5b — EM-LLM (episodic event segmentation).**
* While reading a long sequence, the model computes per-token **Bayesian surprise**.
* Local maxima of surprise are treated as *event boundaries*; tokens between boundaries form an episodic event chunk.
* Boundaries are then refined via graph-theoretic modularity to produce coherent episodes.
* At retrieval time, two retrieval channels run in parallel: *similarity* (which events are semantically related to the query?) and *temporal contiguity* (which events happened *near* a retrieved event?).
* Demonstrated retrieval over 10 M tokens — far beyond what full-context Transformers can handle — without fine-tuning.

**Strengths.**
* HippoRAG: principled answer to multi-hop retrieval with a single graph traversal. Matches iterative-retrieval accuracy at a fraction of cost.
* EM-LLM: works on raw token streams with no extraction pipeline — pure activation-driven structure. Matches or beats full-context models without their compute cost.

**Weaknesses.**
* HippoRAG: PageRank's quality depends entirely on graph quality, which depends on extraction.
* EM-LLM: surprise-based boundaries are model-dependent; episodes are *spans of tokens*, so multi-modal or multi-document settings are awkward.

**Why it's its own concept.** The defining choice is *neuroscience-informed retrieval*. C2 retrieves the nearest point; C3 traverses an explicit graph; C5 simulates a brain region — Personalized PageRank as a stand-in for hippocampal pattern completion (HippoRAG), or surprise-driven segmentation as a stand-in for episodic event cognition (EM-LLM).

---

## Part 3 — How the Five Concepts Compare

| Axis | C1 OS-paged | C2 Extracted vectors | C3 Temporal KG | C4 Self-organizing | C5 Bio-inspired |
|---|---|---|---|---|---|
| Storage substrate | Vector DB + scratchpad | Vector DB | Graph DB (typed) | Vector DB + emergent links | Graph (HippoRAG) / KV cache (EM-LLM) |
| Memory granularity | Free-form notes | Atomic facts | Entities + relations | Linked notes | Subgraphs / event chunks |
| Update policy | Agent-driven | Overwrite + dedupe | Bi-temporal invalidation | Link rewiring + context update | Append (HippoRAG) / online (EM-LLM) |
| Retrieval primitive | LLM tool call | Cosine similarity | Hybrid: BM25 + cosine + traversal | Similarity + link expansion | Personalized PageRank / surprise + contiguity |
| Time model | None native | Timestamps as metadata | Bi-temporal first-class | Implicit via reinterpretation | Temporal contiguity (EM-LLM) |
| Forgetting | None (manual) | Forgetting curve / overwrite | Invalidation, never delete | Link decay / rewrite | None / sliding |
| Best at | Open-ended agents, tool ecosystems | Single-hop fact recall, low latency | Multi-hop, time-travel queries, knowledge updates | Reinterpretive contexts, evolving understanding | Multi-hop retrieval (HippoRAG); long-context QA (EM-LLM) |
| Weakest at | Cost, reliability of self-management | Multi-hop, temporal | Schema brittleness, extraction cost | Write-time cost, less production-tested | Single benchmark; pipeline-specific |
| Status | Production (Letta) | Production (Mem0, OpenAI) | Production (Zep, Cognee) | Research (NeurIPS 2025) | Research (NeurIPS 2024 / ICLR 2025) |

## Part 4 — Open Questions and Where the Field Is Heading

1. **Selective forgetting is unsolved.** MemoryAgentBench was constructed specifically because no system handles forgetting + retrieval + reasoning + test-time learning at once. C2 has shallow forgetting (curves), C3 has invalidation but never deletes, C1/C4/C5 have essentially none.
2. **Extraction is the bottleneck.** Every system except C1-MemGPT and C5b-EM-LLM depends on an LLM extraction pass at write time. Quality of memory is upper-bounded by quality of extraction.
3. **No shared harness.** ByteRover, Hindsight, Supermemory, Mem0, Zep all publish self-reported numbers. Independent harnesses (Maximem Synap, AMB) are emerging but the field still lacks a Kaggle-style canonical leaderboard.
4. **Hybrids are winning in practice.** Cognee, Mem0-Graph, Zep all combine ≥2 of the above concepts. The pure-form systems are conceptual reference points; production systems blend.
5. **A 6th concept on the horizon: parametric memory.** Cartridges (Stanford Hazy Research, 2025), sparse memory finetuning, and KV-cache-as-memory approaches treat memory as model weights or compressed caches. Not yet on the agent-memory leaderboards but a credible threat to all five categories above — especially C2 and C5b.

---

## Sources

### Benchmark papers and pages
- [LongMemEval (ICLR 2025) — paper](https://arxiv.org/abs/2410.10813)
- [LongMemEval — GitHub](https://github.com/xiaowu0162/LongMemEval)
- [LongMemEval — project page](https://xiaowu0162.github.io/long-mem-eval/)
- [LoCoMo (ACL 2024) — paper](https://arxiv.org/abs/2402.17753)
- [LoCoMo — project page](https://snap-research.github.io/locomo/)
- [LoCoMo — GitHub](https://github.com/snap-research/locomo)
- [MemoryAgentBench (ICLR 2026) — paper](https://arxiv.org/abs/2507.05257)
- [MemoryAgentBench — GitHub](https://github.com/HUST-AI-HYZ/MemoryAgentBench)
- [Letta Leaderboard](https://docs.letta.com/leaderboard)
- [Letta Leaderboard blog post](https://www.letta.com/blog/letta-leaderboard)
- [Agent Memory Benchmark (AMB)](https://agentmemorybenchmark.ai/)

### System papers and references
- [MemGPT (arXiv 2310.08560)](https://arxiv.org/abs/2310.08560)
- [Letta docs — memory management](https://docs.letta.com/advanced/memory-management/)
- [Mem0 (arXiv 2504.19413)](https://arxiv.org/abs/2504.19413)
- [Mem0 — GitHub](https://github.com/mem0ai/mem0)
- [Zep (arXiv 2501.13956)](https://arxiv.org/abs/2501.13956)
- [Graphiti — GitHub](https://github.com/getzep/graphiti)
- [Cognee — GitHub](https://github.com/topoteretes/cognee)
- [A-MEM (arXiv 2502.12110)](https://arxiv.org/abs/2502.12110)
- [A-MEM — GitHub (NeurIPS 2025)](https://github.com/WujiangXu/A-mem)
- [HippoRAG (NeurIPS 2024, arXiv 2405.14831)](https://arxiv.org/abs/2405.14831)
- [EM-LLM (arXiv 2407.09450)](https://arxiv.org/abs/2407.09450)
- [EM-LLM — project page](https://em-llm.github.io/)

### Comparative writeups
- [Zep's "State of the Art" comparison](https://blog.getzep.com/state-of-the-art-agent-memory/)
- [Mem0 — graph memory comparison](https://mem0.ai/blog/graph-memory-solutions-ai-agents)
- [State of AI Agent Memory 2026 (Mem0)](https://mem0.ai/blog/state-of-ai-agent-memory-2026)
- [Letta — Benchmarking AI Agent Memory](https://www.letta.com/blog/benchmarking-ai-agent-memory)
- [ByteRover LongMemEval results](https://www.byterover.dev/blog/benchmark-ai-agent-memory)
- [Cognee benchmark vs Mem0 / Graphiti / LightRAG](https://www.cognee.ai/blog/deep-dives/ai-memory-evals-0825)
- [Cartridges — Hazy Research](https://hazyresearch.stanford.edu/blog/2025-06-08-cartridges)
- [Awesome-Memory-for-Agents reading list](https://github.com/TsinghuaC3I/Awesome-Memory-for-Agents)
