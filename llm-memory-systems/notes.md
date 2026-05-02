# LLM/Agentic Memory Systems Research Notes

## Goal
1. Identify the best benchmarks for LLM/agentic memory: how do they work, what dataset, what measurements
2. Find the top 5 systems on benchmark leaderboards
3. Deep dive into each system, grouping by *concept* (not execution)

## Working notes (chronological)

### Benchmark survey

**LongMemEval** (ICLR 2025, Wu et al., arxiv 2410.10813)
- 500 manually created questions embedded in scalable chat histories
- 5 core memory abilities tested: information extraction, multi-session reasoning, temporal reasoning, knowledge updates, abstention
- 6 question types: single-session-user, single-session-assistant, single-session-preference, temporal-reasoning, knowledge-update, multi-session
- 3 sizes: LongMemEval_S (~115K tokens), LongMemEval_M (500 sessions, ~1.5M tokens), LongMemEval_oracle (just gold-evidence)
- Decomposes long-term memory pipeline into 3 stages: indexing, retrieval, reading
- Metrics: GPT-4o LLM-as-judge (>97% agreement with humans), Recall@k, NDCG@k for retrieval
- Key finding: GPT-4o drops from 91.8% (offline full read) to 57.7% (online setting). 30% drop typical for commercial assistants.
- Abstention questions (suffix `_abs`) seek info NOT in history -> agent should say "I don't know"

**LoCoMo** (ACL 2024, Maharana et al., Snap Research, arxiv 2402.17753)
- ~600 turns, ~16K tokens average per conversation, up to 32 sessions
- Built via machine-human pipeline using LLM agents grounded on personas + temporal event graphs; humans verify
- Tasks: question answering, event summarization, multi-modal dialogue generation
- QA categories: single-hop, multi-hop, temporal reasoning, open-domain knowledge, adversarial
- Each agent can share/react to images (multi-modal)
- Has become the primary testbed for memory systems

**MemoryAgentBench** (ICLR 2026, Hu et al., arxiv 2507.05257)
- 4 core competencies: accurate retrieval, test-time learning, long-range understanding, selective forgetting
- Repurposes long-context datasets into multi-turn incremental format
- New datasets: EventQA (retrieval), FactConsolidation (conflict resolution)
- Tests context-RAG, external memory modules, tool-integrated agents
- Finding: no current method masters all 4 competencies

**Letta Leaderboard** (May 2025, Letta team)
- Evaluates LLMs (not memory systems) on agentic memory ops while keeping framework constant
- Measures core memory (in-context) + archival memory (external) on read/write/update
- Top: Claude 4 Sonnet, GPT-4.1, GPT-4o; cost-effective: Gemini 2.5 Flash, GPT-4o-mini

**DMR (Deep Memory Retrieval)** - Older benchmark used to compare MemGPT/Zep. Zep 94.8% vs MemGPT 93.4%.

### Reported scores (LongMemEval-S, GPT-4o reader unless noted)
Note: scores vary because of harness, model used, prompts. Take them as directional.

- ByteRover 2.0: 92.2% (self-reported)
- Hindsight: 89.6%
- Supermemory: 85.4%
- Memobase: 75.8%
- Zep: 75.1% (some report 71.2% from paper)
- Mem0: 66.9% (66.9% LLM-judge on LoCoMo vs OpenAI 52.9%)
- OpenAI Memory: 52.9%
- Maximem Synap: 90.2%
- MemPalace: claimed 96.6%

### Conceptual categories identified (de-duplicated from execution)

C1. **OS-paged hierarchical memory**
   - MemGPT (arxiv 2310.08560) / Letta - virtual context management. Core (RAM) vs archival (disk). Self-edit via function calls.

C2. **Extractive vector-based fact memory**
   - Mem0 (vanilla) - LLM extracts atomic facts -> dense embeddings -> ANN search
   - MemoryBank - daily summaries + Ebbinghaus forgetting curve
   - OpenAI Memory - similar paradigm, simpler
   - Generative Agents (Park et al.) - recency × relevance × importance scoring

C3. **Temporal knowledge graph memory**
   - Zep/Graphiti (arxiv 2501.13956) - bi-temporal edges (t_valid/t_invalid), hybrid BM25+semantic+graph
   - Mem0-Graph - entity-relation triplets in graph DB (FalkorDB)
   - Cognee - graph+vector hybrid, 14 retrieval modes, GraphRAG
   - LightRAG - lighter GraphRAG

C4. **Self-organizing agentic memory (Zettelkasten-style)**
   - A-MEM (NeurIPS 2025, arxiv 2502.12110) - notes with attributes, dynamic linking, memory evolution
   - Reflective Memory Management (RMM) - prospective + retrospective reflection
   - H-MEM - hierarchical memory layers

C5. **Biologically-inspired episodic / hippocampal memory**
   - HippoRAG (NeurIPS 2024, arxiv 2405.14831) - LLM->KG, Personalized PageRank for retrieval, mimics neocortex+hippocampus
   - EM-LLM (arxiv 2407.09450) - Bayesian-surprise event boundaries, similarity + temporal-contiguity retrieval, scales to 10M tokens

(Bonus C6 - parametric/in-weights: Cartridges, sparse memory finetuning, LoRA continual learning. Less mature for production agents but conceptually distinct.)

### Picking the top 5 by concept

The 5 conceptual buckets I'll cover (chosen because each represents a fundamentally different answer to "where do memories live and how do you find them?"):

1. **OS-paged hierarchical memory** (MemGPT/Letta)
2. **Extractive vector-based fact memory** (Mem0, MemoryBank, OpenAI Memory, Generative Agents)
3. **Temporal knowledge graph memory** (Zep/Graphiti, Cognee, Mem0-Graph)
4. **Self-organizing agentic memory** (A-MEM, RMM)
5. **Biologically-inspired episodic memory** (HippoRAG, EM-LLM)

### Key design axes for comparison
- Storage substrate: KV cache / vector DB / graph DB / hybrid
- Memory granularity: tokens / facts / entities+edges / notes / events
- Update policy: append-only / overwrite / forgetting curve / temporal invalidation / link evolution
- Retrieval primitive: similarity / BM25 / graph traversal / PageRank / temporal contiguity
- Who controls writes: implicit pipeline / agent self-edit via tools / hybrid
- Time awareness: timestamps only / bi-temporal / forgetting curve / event graph
