# The Geometry of Forgetting — Research Report

A focused read of Ashwin Gopinath's LinkedIn pulse article *"The Geometry
of Forgetting: Why Brains and LLMs Fail in Exactly the Same Way"* and
the supporting paper trilogy, with emphasis on what it means for **LLM
and agent memory** design.

> Source the user pointed at:
> https://www.linkedin.com/pulse/geometry-forgetting-why-brains-llms-fail-exactly-same-ashwin-gopinath-ppsac/
>
> The LinkedIn page itself, the matching X thread, the Substack
> companion, and the arXiv / OpenReview PDFs all rejected unauthenticated
> fetches. The reconstruction below uses the publicly readable GitHub
> README (`niashwin/geometry-of-consolidation`) plus quoted fragments
> surfaced by web search across all of those sources. See `notes.md` for
> the search trail.

## 1. Who is making the argument

- **Ashwin Gopinath** — former MIT (bionanotech), now co-founder of
  **Sentra.app**, an enterprise-memory startup ($5M seed, a16z
  speedrun + Together Fund, customer SoftBank). Writes the
  *Nano Thoughts* Substack.
- The LinkedIn piece is the popular framing of a three-paper sequence:
  1. **The Price of Meaning: Why Every Semantic Memory System Forgets**
     — arXiv 2603.27116 (Mar 2026). The *No-Escape Theorem*.
  2. **The Geometry of Forgetting** — OpenReview `qKh7Aip3JC` (the
     "HIDE" preprint in the GitHub README). Empirical replication of
     human memory phenomena from off-the-shelf embeddings.
  3. **The Geometry of Consolidation** — NeurIPS 2026 submission;
     `github.com/niashwin/geometry-of-consolidation`. Derives the
     central inequality and the GAC algorithm.
- Companion Substack essays: *Why AI Needs to Forget*, *The Art of
  Forgetting*, *Memory Is State, Not a Service*, *Company Brain* (parts
  1–2).

> Disambiguation: a different group authored arXiv 2605.09195 ("The
> Geometry of Forgetting: Temporal Knowledge Drift as an Independent Axis
> in LLM Representations"). Same title, unrelated topic. Don't conflate.

## 2. The thesis

Forgetting in vector-based / embedding-based memory systems — RAG, agent
long-term memory, episodic stores — is **not a bug, not a context-window
artifact, and not fixable by buying more storage**. It is a *geometric
inevitability* of any retrieval system that

1. organises information by **meaning** (continuous semantic space),
2. retrieves by **proximity** (cosine / inner-product similarity), and
3. has **finite effective dimensionality**.

Run unmodified human-memory paradigms through a stock pretrained encoder
with **no parameter fitting** and you get out:

- the Ebbinghaus **power-law forgetting curve**,
- DRM **false memories** at near-human rates,
- the **spacing effect** with the right ordering,
- **tip-of-tongue** states.

The match isn't because anyone modelled biology — it falls out of the
geometry. Conversely: human forgetting exponents sit in the regime the
geometry predicts for moderate effective dim. So the headline rhetorical
move is: *brains and LLMs share a failure mode because they share a
representational format, not because either copied the other.*

## 3. The "dimensionality illusion" — the most actionable finding

Production embedding models advertise 384 / 768 / 1024 dimensions. Run
PCA: ~95% of the variance fits in **~16–18 effective dimensions**.

- MiniLM (nominal 384) → effective dim ≈ **16**.
- BGE-large at 64 PCA dims actually outperforms MiniLM at 384 nominal,
  because *effective* dim is what governs interference.

The mapping from effective dim to power-law forgetting exponent `b`
(competitor count fixed at 40k same-article distractors):

| effective d | forgetting exponent b | regime              |
|-------------|-----------------------|---------------------|
|  64         |  ≈ 0.161              | strong interference |
| 128         |  ≈ 0.020              | mild                |
| ≥ 256       |  < 0.004              | effectively immune  |

Drop the competitors → exponent falls **~50×**. The implication is
sharp: **forgetting is interference, not decay**. Memories don't fade;
they get crowded out in low-effective-dim space.

## 4. Quantitative replications of human results (no tuning)

Run on a 1024-dim retrieval model, raw cosine similarity, no fits:

- **DRM false-memory rate** (24 published lists): critical-lure false
  alarms = **0.583** vs human ≈ 0.55 — within 3.3 pp at zero unrelated
  false alarms.
- **Tip-of-tongue rate**: **3.66 ± 0.13%** vs human ≈ 1.5% — right
  direction, ~2× off. Interpreted as biology having extra stabilising
  machinery on top of geometry.
- **Spacing effect**: ordering matches human exactly
  `long > medium > short > massed`, with Cohen's d ≈ **13.1**
  separating spaced from massed. (Effect size that large suggests very
  low within-condition variance in the model run — see Caveats.)

## 5. The central inequality (from the consolidation paper)

  ε_id ≥ 1 − c₁ · m · (θ' / d̄)^(d_eff / 2)

- `d̄` = mean within-cluster distance,
- `θ'` = retrieval threshold,
- `d_eff` = effective dimension,
- `m` = number of representative vectors kept,
- `ε_id` = identity-preservation error.

Two regimes:

- **Tight** (`d̄ < θ'`): identity is preservable cheaply; one centroid
  per cluster is fine.
- **Spread** (`d̄ ≥ θ'`): the bound flips — compression *must* lose
  identity. This is the **No-Escape Theorem**: any meaning-organised
  memory under mild axioms obeys a strict capacity ↔ identity tradeoff.

The derived algorithm — **GAC, Geometry-Aware Consolidation** — picks
centroid in the tight regime and a residual-budgeted medoid in the
spread regime. Pareto-dominates 8 baselines across 6 encoders × 7
corpora.

## 6. What this means for LLM / agent memory

This is the part most relevant to the user's question. The piece is
essentially a sustained argument against naïve vector-DB-as-memory.

1. **Vector-store memory degrades as a power law in corpus size.** It
   is not a hyperparameter tuning issue; the exponent is set by the
   encoder's effective dimensionality. Long-running agents with
   ever-growing memory will hit this.
2. **Nominal embedding dim is misleading.** A 1536-dim OpenAI vector
   isn't 1536-dim where interference cares. Audit the spectrum.
3. **The same geometry that makes embeddings useful for analogy is
   what produces false recall.** You cannot have semantic
   generalization *and* perfect identity. Choose your point on the
   Pareto curve.
4. **Across five architectures tested** — vector retrieval, graph
   memory, attention retrieval, BM25 filesystem, parametric memory —
   pure semantic retrievers show the geometric symptoms directly.
   Reasoning-equipped systems can paper over them behaviorally, but at
   the cost of converting *smooth degradation* into *brittle, sudden
   failure*. (Worse for ops: harder to detect and harder to fix.)
5. **Practical levers**:
   - Increase **effective** dim — whitening, decorrelation losses,
     orthogonalization at index time.
   - **Hybrid retrieval** — bolt BM25 / keyword channel alongside
     embeddings; sparse and dense fail in uncorrelated ways.
   - **Explicit consolidation** — GAC-style routing: route each
     cluster to the consolidator its geometry requires.
   - **Graph layer over vectors** — lets unreinforced edges decay.
     This is the "memory is a *graph*, not a *database*" framing from
     *Why AI Needs to Forget*: a graph is good at memory precisely
     because it is good at *forgetting*.
6. **The graph + vector hybrid (Sentra's product framing)** reports
   ~**52% Recall@5** vs ~25% for a stateless vector store, with ~84%
   fewer tokens consumed.

## 7. The wider Gopinath narrative — why this is a product, not a paper

The Substack series ties the geometric claim to a business thesis:

- *Why AI Needs to Forget*: forgetting is not failure, it is lossy
  compression doing its job. Biological memory is a graph because
  graphs *forget well*.
- *The Art of Forgetting*: an agent that remembers every detail can't
  generalize. The next breakthrough is "forgetting well," not
  "remembering more."
- *Memory Is State, Not a Service*: every per-tool memory silo
  (meeting recorder, vector index, agent scratchpad) is its own
  forgetting curve. The org needs *shared state* with exact retrieval,
  semantic retrieval, graph traversal, and state-change-as-query — not
  a federation of memory APIs.
- *Company Brain*: Sentra's pitch — a shared semantic state across
  comms, docs, and agent traces, with one consolidation policy rather
  than N inconsistent ones.

So the LinkedIn piece is doing double duty: a real technical result
(the No-Escape / interference-driven forgetting) and a positioning
argument for Sentra's organizational-memory product.

## 8. Critical assessment

- The match between brains and LLMs is real and direction-correct, but
  "EXACTLY the same way" is rhetorical. DRM is within 3 pp, tip-of-
  tongue is ~2× off. The honest claim is *same regime, same
  qualitative signatures, quantitatively close on some axes*.
- **Cohen's d = 13.1** for spacing is anomalously large. Model runs
  often have tiny within-condition variance because they are nearly
  deterministic; that inflates standardized effect sizes. The
  *ordering* result (long > medium > short > massed) is the more
  interesting finding.
- The "16 effective dimensions" number depends on the variance-
  threshold choice (95%). At 99% the number is larger. The qualitative
  point — effective ≪ nominal — survives. Specific integers don't.
- Geometric sufficiency ≠ mechanistic identification. Showing geometry
  alone reproduces human signatures is consistency evidence, not proof
  that brains use the same mechanism.
- The "no-escape" framing is strongest under the stated axioms
  (kernel-threshold retrieval, monotone-in-inner-product scoring,
  finite intrinsic dim). Systems that score over **structured**
  representations — e.g. propositional / symbolic memories, MoR-style
  routed memories, episodic stores keyed on time — sit outside the
  theorem's scope. The framing acknowledges this; readers sometimes
  don't.
- Strongest practical takeaway: when designing agent memory, **measure
  the effective dimensionality of your encoder** and treat it as a
  first-class design variable, not the advertised number.

## 9. Bottom line

If you build LLM agents with vector-store memory, this piece is worth
internalising not for the brain analogy but for two concrete claims:

1. *Effective* embedding dim, not nominal, controls how fast your
   memory degrades — and most production encoders sit at ~16, well
   inside the interference-vulnerable regime.
2. Forgetting is fundamentally **interference between competing
   memories**, not temporal decay. So adding storage doesn't help;
   adding orthogonality, hybrid retrieval, or geometry-aware
   consolidation does.

The brain comparison is the headline. The dimensionality audit and the
GAC consolidation framing are the lessons that actually change how you
build.

## Sources

- LinkedIn pulse (original, not fetchable unauthenticated):
  https://www.linkedin.com/pulse/geometry-forgetting-why-brains-llms-fail-exactly-same-ashwin-gopinath-ppsac/
- X thread:
  https://x.com/ashwingop/status/2042091130213560759
- The Geometry of Consolidation (GitHub, fetched directly):
  https://github.com/niashwin/geometry-of-consolidation
- The Price of Meaning (arXiv abstract page):
  https://arxiv.org/abs/2603.27116
- The Geometry of Forgetting (OpenReview PDF):
  https://openreview.net/pdf?id=qKh7Aip3JC
- Why AI Needs to Forget (Substack):
  https://nanothoughts.substack.com/p/why-ai-needs-to-forget
- The Art of Forgetting (Substack):
  https://nanothoughts.substack.com/p/the-art-of-forgetting-how-ai-learns
- Memory Is State, Not a Service (Substack):
  https://nanothoughts.substack.com/p/memory-is-state-not-a-service
- Company Brain (Substack):
  https://nanothoughts.substack.com/p/company-brain-why-most-companies
- Sentra.app funding coverage (background):
  https://pulse2.com/sentra-5-million-seed-funding/
