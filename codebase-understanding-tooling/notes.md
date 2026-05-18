# Research Notes: Tooling for Codebase Understanding & Testability

## Goal
Survey the landscape of tools, techniques, articles, and approaches for:
1. Understanding existing codebases (especially large/legacy ones)
2. Reviewing code
3. Introducing tests into untested code (legacy code)

## Reference starting points (from the user)
- feenk.com — Glamorous Toolkit / "moldable development"
- legacycode.com / understandlegacycode.com — Michael Feathers / Nicolas Carlo
- "Software Design X-Rays" by Adam Tornhill
- "Your Code as a Crime Scene" by Adam Tornhill (CodeScene)

## Research approach
Direct WebFetch was 403-blocked for most domains. Used WebSearch (Google-indexed
snippets) which returned solid summaries. Cross-linked each tool/idea to its
primary source. Notes below are organised by family.

---

## Family A — "Behavioral code analysis" school (Adam Tornhill)

Idea: a codebase is not just a static artifact — it has a *history* in the
version control system. By mining commits you can discover hotspots
(complex code that also changes a lot), temporal/change coupling (files
that change together implicitly), knowledge maps, and team / Conway's-Law
patterns.

Books:
- "Your Code as a Crime Scene" (1st ed 2015, 2nd ed 2024 with Adam Tornhill +
  Adam Petersen).
- "Software Design X-Rays" (Pragmatic, 2018) — more advanced techniques:
  hotspots, change coupling, refactoring patterns, social side of code,
  Conway's Law, long-term complexity trends. Key insight: "change alone is
  the single most important metric when it comes to quality issues."

Tools:
- code-maat — open source CLI (Clojure JAR), mines git/hg/svn/p4/tfs.
  Computes coupling, churn, code age, etc. https://github.com/adamtornhill/code-maat
- CodeScene — commercial successor, founded 2015 by Tornhill. Combines
  VCS data + ML. Detects hotspots, change coupling, code health (cognitive-
  complexity flavored), author-churn risk, knowledge loss, even predicts
  security vulnerabilities. Plays vs static analysis because it adds the
  *temporal* dimension.
- code-forensics — OSS toolset by smontanari that builds on Tornhill's ideas.

Key blog: understandlegacycode.com has a nice
"key points of Software Design X-Rays" summary.

---

## Family B — Michael Feathers / Working Effectively with Legacy Code (WELC)

Definition: "legacy code = code without tests" (Feathers).

Core concepts:
- Seams: a place where you can change behavior without editing in that place.
  Types: preprocessor seams, link seams, object seams.
- Characterization tests: tests that pin down what code currently does
  (not what it *should* do). Written by running code, capturing output, and
  asserting on that captured output. Becomes a safety net for refactoring.
- Algorithms catalog: Sprout Method/Class, Wrap Method/Class, Extract and
  Override Call, Subclass and Override Method, Pass Null, etc.

Why characterization first? Because users depend on actual behavior, not
intended behavior.

---

## Family C — Approval / Golden-master testing (Llewellyn Falco, Emily Bache)

Approval testing = mechanise characterization tests. The library captures
the output of code into an "approved" file; runs compare against it.
Workflow: Arrange → Act → Print → Compare.

Libraries:
- ApprovalTests.* — polyglot: .NET, Java, Node, PHP, Python, Ruby, C++, Go,
  Perl, Objective-C. approvaltests.com
- TextTest (Emily Bache)
- Verify (.NET, by Simon Cropp) — modern competitor.

Emily Bache also maintains the canonical refactoring katas
(emilybache/GildedRose-Refactoring-Kata, ported to 40+ languages) used to
teach these techniques.

---

## Family D — Refactoring catalogs & method

- Martin Fowler, "Refactoring" 2nd ed (2018) — JS examples, ~70 refactorings,
  15 new ones. Less class-centric than 1st ed. https://refactoring.com
- Joshua Kerievsky, "Refactoring to Patterns".
- Mikado Method (Ola Ellnestam & Daniel Brolund, Manning 2014) — build a
  dependency graph of refactor steps by attempting changes, reverting,
  and recording the "I need this first" arrows. Then execute leaves up.
- Strangler Fig pattern (Martin Fowler, 2004) — incrementally route traffic
  from the old system to new pieces behind a facade until the old is dead.

---

## Family E — Static-analysis / code-quality platforms

Open source / free tier:
- SonarQube / SonarCloud — 6000+ rules, code smells, cognitive complexity,
  tech-debt minutes, A-E maintainability rating.
- Code Climate (similar concept, hosted).

.NET / Java / multi-language specialised:
- NDepend (.NET) — CQLinq queries, dependency graph & matrix, cycle
  detection, architectural rules.
- Structure101 — DSM, layering, complexity, cycle breaking. Java/.NET/C++.
- Lattix — DSM editing, architectural metrics. Highly enterprise-y.
- SciTools Understand — most thorough reporting; safety-critical / DO-178C
  / MISRA. Onboarding-oriented graphs.

Big enterprise platforms:
- CAST Highlight — application portfolio scans, cloud-readiness, M&A.
- SIG Sigrid — benchmarked against 25 000 systems.
- Fortify / Checkmarx / Veracode — security-leaning.

---

## Family F — Architecture-as-tests / fitness functions

- ArchUnit (Java) — write JUnit tests that assert dependency, layering,
  cycle, naming rules. Can ingest PlantUML diagrams as the spec.
- Deptrac (PHP) — YAML config, layer rules, CI integration, Graphviz/
  Mermaid output.
- jdeps (built into JDK 8+) — class/JAR dependency analysis, dot output.
- NetArchTest (.NET).
- Konsist (Kotlin).

These embody Neal Ford / Rebecca Parsons' "Building Evolutionary
Architectures" idea: fitness functions that fail the build when
architectural invariants are violated.

---

## Family G — Structural search / large-scale refactoring (codemods)

- jscodeshift (Facebook) — JS/TS AST transforms.
- OpenRewrite (Java/Kotlin/etc; Lossless Semantic Trees, recipe catalog) +
  Moderne (commercial accelerator).
- comby — language-agnostic structural search/replace with hole templates,
  not AST-aware.
- Semgrep — AST patterns, security/lint focus, big rule library.
- ast-grep — Rust, tree-sitter based, fast polyglot, query+rewrite.
- ts-morph — programmatic TS AST manipulation.
- Codemod (Meta), CodemodAI (codemod.com).

Trend: "codemods + LLM" — Semgrep AutoFixes use LLMs; OpenRewrite/Moderne
applies recipes across thousands of repos at once.

---

## Family H — Semantic / variant analysis

- CodeQL (GitHub) — query your code like a database (Datalog-ish). Used for
  security-research style "variant analysis" — find all variants of a known
  bug across 1000+ repos (MRVA from VS Code).
- Glean (Meta) — fact-based code-knowledge graph.
- Stack Graphs (GitHub) — incremental, name-resolution graphs.
- LSIF / SCIP — index formats consumed by tools like Sourcegraph.

---

## Family I — Visual / 3D / "city" code maps

- CodeCity (Wettel, U Lugano) — classes as buildings, packages as
  districts. Research origin of the metaphor.
- CodeCharta (MaibornWolff, OSS) — production-ready treemap city visual,
  imports from Sonar, Tokei, code-maat, etc. Even 3D-printable. Supports
  delta comparison.
- Gource — animation of repo history through time.
- git-of-theseus (Erik Bernhardsson) — stacked-area plot of "how much code
  written in year X still survives".
- CodeViz (VS Code extension), Emerge, repostats.

---

## Family J — Onboarding / navigation / interactive maps

- Sourcetrail — once-popular OSS, now abandoned (last release 2021) but
  still useful for C/C++/Java/Python interactive code map.
- CodeSee — maps + tours, code ownership overlays, onboarding-first.
- Swimm — docs co-located with code, auto-updates docs when code changes.
- Sourcegraph — universal code search, references, batch changes,
  "Code Insights" trends.
- Sourcegraph Cody — RAG over your repos for chat/explain.
- Cursor / Aider / Cline / Claude Code — agentic AI in your editor or
  terminal that traverses repos with tree-sitter or repo-maps.

---

## Family K — Moldable development (feenk / Glamorous Toolkit)

Tudor Gîrba & feenk. Thesis: don't read source code — *mould* the
environment around the question you have. Build many tiny, throwaway,
contextual tools per project to make a system "explainable".

Components:
- Glamorous Toolkit (GT) — Pharo-based moldable environment. Now v1.0
  after 6 years of work + 14 years of research.
- GT Inspector — every object has multiple custom "views". Open a
  collection and you might see "Items / Sum / SQL query / Map".
- GT Documenter — live notebooks (Lepiter) where code snippets execute
  inline. Mixed prose + executable + visualisations.
- GT Coder — expandable editors, examples-as-tests.
- Bloc — graphics framework.
- Pharo + Moose (older, separate but related) — for big software /
  data analyses; FAMIX metamodel, language-independent reverse
  engineering. Synectique consultancy uses these for legacy audits.

---

## Family L — Mutation testing (proof your tests actually test)

- PIT / PITest (Java) — most mature. Maven/Gradle/IDE integration.
- Stryker — JS/TS, C#, Scala.
- mutmut, cosmic-ray (Python).
- mull (LLVM / C/C++).
- Microsoft has a built-in mutation-testing pipeline for .NET now.

Score = % mutants killed. More honest than line coverage.

---

## Family M — Property-based & generative testing

- QuickCheck (Haskell) — original. Generators + shrinking.
- Hypothesis (Python) — wide industry adoption.
- fast-check (JS/TS), jqwik (Java), Hedgehog (Haskell/F#/Scala).
- Especially useful for legacy code: feed random structured input, assert
  invariants (round-trip equals, idempotency, etc.) to flush out bugs the
  original author never thought of.

---

## Family N — Architecture documentation & decision capture

- C4 model (Simon Brown, 2006-2011) — 4 levels: Context, Containers,
  Components, Code. https://c4model.com
- Structurizr — Simon Brown's tooling. Models-as-code DSL, version-
  controlled, interactive zoom/animate. Generates from one model into
  PlantUML, Mermaid, draw.io, etc.
- ADRs (Architecture Decision Records) — Michael Nygard's pattern. One
  short markdown per decision. adr.github.io. Toolchain: adr-tools (Nat
  Pryce), Log4brains, MADR template.
- Living Documentation (Cyrille Martraire, 2019) — make the source of
  truth executable; tests/code generate the doc.
- Event Storming (Alberto Brandolini) — large sticky-note workshop;
  classic technique to recover business logic from a legacy system.
- Domain Storytelling (Stefan Hofer & Henning Schwentner) — smaller, more
  moderated; tells "the actor does X with Y" stories.

---

## Family O — Runtime / dynamic understanding

- Brendan Gregg's flame graphs — sampled-stack visualisation; understand
  *where the program actually spends time*. github.com/brendangregg/FlameGraph
- eBPF tooling (bpftrace, bcc) — production-safe tracing.
- OpenTelemetry traces + service maps (Jaeger, Tempo, DataDog) — the
  microservice-era "what calls what at runtime".
- Reverse-engineering profilers (py-spy, async-profiler, perf).
- rr (Mozilla) — record & replay debugger; deterministic time-travel.

---

## Nicolas Carlo's understandlegacycode.com (concentrated picks)

Carlo (formerly Software Crafters Montreal) writes the most accessible
day-to-day blog on this topic.

His "Legacy Code: First Aid Kit" e-book = 14 techniques:
1. ADRs
2. Brain Dump (just write what you currently know/believe about a piece
   of code; reveals gaps fast)
3. Over-committing (commit tiny so you can throw work away)
4. Mikado Method
5. Hotspots Analysis
6. Approval Testing
7. Coding Katas (sharpen on safe ground)
8. Characterization tests
9. Working with seams
10. Strangler fig
11. Documenting the why (ADR-adjacent)
12. ... (etc. — 11 are language-agnostic, a few are static-typing only)

Recommended talks (his "3 must-watch talks"):
- "All the Little Things" by Sandi Metz
- "Therapeutic Refactoring" by Katrina Owen
- "Scaling React Server-Side Rendering" by Talia Nassi (talk about pulling
  perf wins from a legacy SSR setup) — varies per post; the consistent
  three he cites are Metz, Owen, and Llewellyn Falco's approval testing
  talk.

---

## Cross-cutting / "best articles" shortlist

- M. Feathers, "The Deep Synergy Between Testability and Good Design"
- M. Fowler, "Strangler Fig Application" (martinfowler.com/bliki/StranglerFigApplication.html)
- M. Fowler, "ArchitectureDecisionRecord" bliki entry
- A. Tornhill, "Embrace your code's evolution" (Pragmatic mag)
- A. Tornhill, "Prioritise Technical Debt as a Business Investment"
- Sandi Metz, "All the Little Things" (RailsConf 2014)
- Kent Beck, "Tidy First?" (2023 book) — micro-refactor before feature.
- Joshua Kerievsky, "Sufficient Design"
- Andrew Stellman / Jennifer Greene, "Head First C# Legacy" chapters
- Shopify Engineering, "Refactoring Legacy Code with the Strangler Fig
  Pattern" (2022)
- Martin Fowler, "Refactoring with Codemods to Automate API Changes"
- E. Bache, "Approval Testing — coding-is-like-cooking.info" series
- Sourcegraph blog, "Anatomy of a coding assistant" (semantic-search
  primer)

---

## Practical "starter recipe" assembled from the above

1. *Mine VCS*: run code-maat or CodeScene Community → list hotspots.
2. *Map dependencies*: ndepend / Structure101 / SciTools / ArchUnit-style
   tests; surface cycles.
3. *Visualize for the team*: CodeCharta (treemap city), Gource (history),
   C4 + Structurizr for the macro picture.
4. *Pick the top hotspot*. Wrap with approval/characterization tests until
   green.
5. *Mikado* the refactor; Sprout/Wrap new behavior.
6. *Lock in invariants* with ArchUnit / Deptrac, mutation-test the new
   tests with PIT or Stryker.
7. *Persist understanding* via ADRs + a Living Documentation pass; in
   parallel run a moldable-development / GT or Sourcegraph notebook on
   the surviving complexity.
8. *Long-term*: Strangler Fig pieces out toward a new architecture.

