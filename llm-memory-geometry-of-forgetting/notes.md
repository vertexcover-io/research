# Research Notes — Geometry of Forgetting (Gopinath)

## Task
Research the LinkedIn article: "The Geometry of Forgetting: Why Brains and
LLMs Fail in Exactly the Same Way" by Ashwin Gopinath, focused on
LLM / agent memory implications.

Source URL (target):
https://www.linkedin.com/pulse/geometry-forgetting-why-brains-llms-fail-exactly-same-ashwin-gopinath-ppsac/

## What I tried

1. **WebFetch on the LinkedIn URL** — 403 Forbidden. LinkedIn pulse articles
   reject scrapes that aren't authenticated.
2. **WebFetch on X/Twitter** (https://x.com/ashwingop/status/2042091130213560759) — 403.
3. **WebFetch on web.archive.org snapshots** — blocked at the harness level
   ("Claude Code is unable to fetch from web.archive.org").
4. **WebFetch on the Substack version** (`nanothoughts.substack.com/p/why-ai-needs-to-forget`) — 403.
5. **WebFetch on arXiv abstract / PDF pages** — 403 (arxiv.org also blocks
   the WebFetch user-agent).
6. **WebFetch on openreview.net PDF** — 403.
7. **WebFetch on the GitHub repo README** for
   `niashwin/geometry-of-consolidation` — **succeeded**; this gave the
   cleanest summary of the paper trilogy and the central inequality.
8. **WebSearch fallback** — worked very well. The search-result summarizer
   surfaced direct quotes from the LinkedIn article, the Substack pieces,
   the arXiv preprint ("The Price of Meaning"), and the OpenReview PDF.
   Repeated targeted searches with author + specific keywords (Ebbinghaus,
   16 effective dimensions, Cohen's d 13.1, b=0.161, DRM word lists,
   spacing effect ordering) returned the quantitative claims and phrasing
   I needed.

So: I could not pull the LinkedIn HTML directly, but I reconstructed the
article's content from (a) the matching Substack/paper companion pieces,
(b) the GitHub repo README, and (c) verbatim fragments returned by
WebSearch snippets.

## Who is Ashwin Gopinath?

- Former MIT professor (bionanotech / molecular engineering background).
- Co-founder & technical lead at **Sentra.app** — enterprise "collective
  intelligence" / organizational-memory startup, $5M seed (a16z speedrun
  + Together Fund), customer SoftBank.
- Writes the **Nano Thoughts** Substack
  (https://nanothoughts.substack.com/). Series of essays in early–mid
  2026 on memory, forgetting, and the "Company Brain":
  - *Why AI Needs to Forget* (Jan 14, 2026)
  - *The Art of Forgetting: How AI Learns to Understand Rather Than Memorize*
  - *Memory Is State, Not a Service*
  - *Company Brain: Why Most Companies Have Data But No Memory*
  - *Company Brain, Part 2: Factual Memory*
- LinkedIn pulse "Geometry of Forgetting..." is the popular-audience
  framing of an underlying paper trilogy:
  1. **The Price of Meaning: Why Every Semantic Memory System Forgets**
     (arXiv 2603.27116, Mar 2026) — the *No-Escape* theorem.
  2. **The Geometry of Forgetting** (OpenReview qKh7Aip3JC, "HIDE" in
     the GitHub README) — empirical replication of human forgetting
     phenomena from embedding geometry.
  3. **The Geometry of Consolidation** (NeurIPS 2026 submission;
     github.com/niashwin/geometry-of-consolidation) — derives the GAC
     algorithm and the central inequality below.

> Note: There is a separately authored arXiv paper "The Geometry of
> Forgetting: Temporal Knowledge Drift as an Independent Axis in LLM
> Representations" (arXiv 2605.09195). That is a different group and
> different topic (temporal knowledge drift) — not the Gopinath piece.
> Don't conflate the two.

## Central thesis (the LinkedIn piece)

Forgetting in LLM / vector / agent memory is not an engineering bug, a
quirk of the context window, or a defect of biological wetware. It is a
**geometric inevitability** of any retrieval system that:

1. organises information by *meaning* (continuous semantic embeddings),
2. retrieves by *proximity* (cosine / inner-product similarity), and
3. operates in finite *effective* dimensionality.

Take a stock pretrained encoder, run unmodified human memory paradigms
through it with no parameter fitting, and you reproduce the
Ebbinghaus power-law forgetting curve, the DRM false-memory rate, the
spacing effect (with the correct ordering long > medium > short >
massed), and tip-of-tongue states.

## Headline quantitative claims (collected from searches)

- **Dimensionality illusion**: production encoders advertised at
  384 / 768 / 1024 dims concentrate ~95% of variance in **~16–18
  principal components**. MiniLM (nominal 384) has effective dim ~16.
- **Power-law exponent vs effective dim** (with 40,000 same-article
  competitors):
  - d_eff = 64  → b ≈ 0.161 (strong interference, clean power law)
  - d_eff = 128 → b ≈ 0.020 (mild)
  - d_eff ≥ 256 → b < 0.004 (effectively gone)
  - 50× drop in forgetting exponent when competitors are removed →
    forgetting is *interference*, not *decay*.
- **DRM false memory**: at the threshold producing zero unrelated false
  alarms, critical-lure false-alarm rate = **0.583** (human ≈ 0.55).
  Within 3.3 percentage points, zero tuning, 1024-dim retrieval over
  24 published DRM word lists.
- **Tip-of-tongue rate**: 3.66 ± 0.13% (human ≈ 1.5%). Same direction,
  ~2× off — interpreted as evidence that biology has additional
  stabilising machinery beyond geometry alone.
- **Spacing effect**: ordering matches humans exactly
  (long > medium > short > massed) with **Cohen's d = 13.1** between
  the spaced and massed conditions in the model.

## Central inequality (from the Geometry of Consolidation README)

  ε_id ≥ 1 − c₁ · m · (θ' / d̄)^(d_eff / 2)

Where d̄ = mean within-cluster distance, θ' = retrieval threshold,
d_eff = effective dimensionality, m = number of representatives, ε_id
= identity-preservation error. Splits into:

- **Tight regime** (d̄ < θ'): identity is cheap; centroid suffices.
- **Spread regime** (d̄ ≥ θ'): compression *must* destroy identity —
  the No-Escape Theorem.

GAC = Geometry-Aware Consolidation: pick centroid in the tight regime,
residual-budgeted medoid in the spread regime. Beats 8 baselines across
6 encoders × 7 corpora (Pareto).

## Implications for LLM / agent memory (the LinkedIn payload)

- A pure vector-store / RAG memory **will** degrade as a power law
  with corpus size. This is predictable from first principles, not a
  tuning problem.
- Increasing the *nominal* embedding dimension doesn't fix it because
  the *effective* dimension is set by the encoder, not by the index.
- The same geometry that makes embeddings useful for analogy /
  generalization is what produces interference and false recall.
  Choose your trade-off — there is a strict Pareto frontier.
- Five memory architectures were stress-tested: vector retrieval, graph
  memory, attention retrieval, BM25 filesystem retrieval, parametric
  memory.
  - Pure semantic retrievers show the geometric symptoms directly.
  - Architectures with explicit reasoning / symbolic indices can mask
    the symptoms behaviorally but trade smooth degradation for
    brittle failures.
- Practical levers Gopinath lists:
  - raise *effective* dim (orthogonalization, whitening, decorrelation
    objectives on the encoder),
  - hybrid retrieval (BM25 / keyword side-channel),
  - explicit consolidation (GAC-style, route by geometry),
  - graph layer over the vectors so unreinforced edges can decay
    (this is the "memory is a graph, not a database" framing from the
    *Why AI Needs to Forget* essay).
- Reported downstream win for the graph + vector hybrid (Sentra-style
  implementation): **52% Recall@5** vs ~25% for a stateless vector
  store, ~84% reduction in token waste.

## Connection to the broader Gopinath narrative

The LinkedIn piece is the "hook." The actual argument lives in the
arXiv "Price of Meaning" paper (no-escape theorem) and the OpenReview
"Geometry of Forgetting" paper (empirical phenomenon replication). The
*business* argument — why this matters for enterprise AI — is the
Company Brain Substack series and the Sentra pitch: every per-tool
memory silo is a per-tool forgetting curve; you need shared *state*,
not a federation of *services*.

## Open questions / what I'd want to verify

- The Cohen's d = 13.1 figure is dramatic — implausibly so for an
  effect-size statistic. Could be an artifact of treating embedding
  retrieval as a near-deterministic process with tiny within-condition
  variance, which inflates d.
- The "16 effective dimensions" claim depends on the PCA-variance
  threshold (they cite 95% explained variance, ~17–18 components).
  Useful but threshold-sensitive.
- "Brains and LLMs fail EXACTLY the same way" — the tip-of-tongue rate
  is 2× off and the DRM rate is 3 percentage points off. "Exactly" is
  rhetorical; "in the same regime and direction" is fair.
- Causal claim — geometry alone is sufficient for these signatures —
  doesn't rule out that biology arrives at the same signatures via
  different mechanisms. The match is consistency, not identification.
