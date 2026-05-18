# Tooling and Techniques for Understanding Codebases and Improving Testability

A survey of the tools, books, articles, and approaches people have built
to do three closely related things:

1. **Understand** an existing codebase (especially large or legacy).
2. **Review** code with more rigor than line-by-line diff inspection.
3. **Introduce tests** into code that has none, so it can be safely changed.

The work is organised into "families" — clusters of tools/ideas that share
a common philosophy. Each family lists primary sources so you can dive
deeper.

---

## TL;DR — the canonical reading list

| # | Book / Resource | Author | Why |
|---|---|---|---|
| 1 | *Working Effectively with Legacy Code* | Michael Feathers (2004) | Defines the field. Seams, characterization tests, the WELC algorithms. |
| 2 | *Refactoring*, 2nd ed | Martin Fowler (2018) | ~70 named refactorings with mechanics. JS examples. |
| 3 | *Your Code as a Crime Scene*, 2nd ed | Adam Tornhill (2024) | Forensic analysis of git history. Hotspots, change coupling. |
| 4 | *Software Design X-Rays* | Adam Tornhill (2018) | More advanced behavioral analysis, social/Conway dimensions. |
| 5 | *The Mikado Method* | Ola Ellnestam & Daniel Brolund (2014) | Graph-based approach for non-trivial refactors. |
| 6 | *Test Driven Development: By Example* | Kent Beck (2002) | The foundational rhythm: red-green-refactor. |
| 7 | *Tidy First?* | Kent Beck (2023) | Small structural cleanups as a first move, not a yak shave. |
| 8 | *Living Documentation* | Cyrille Martraire (2019) | Docs derived from code; ADRs, BDD, examples-as-doc. |
| 9 | *Building Evolutionary Architectures* | Ford, Parsons, Kua | Fitness functions = architecture-as-tests. |
| 10 | *Domain Storytelling* / *EventStorming* | Hofer & Schwentner / Brandolini | Workshop techniques to recover business logic from legacy systems. |

Primary blog/site: [understandlegacycode.com](https://understandlegacycode.com) — Nicolas Carlo's distilled, very practical write-ups.

---

## Family A — Behavioral Code Analysis (Adam Tornhill school)

> *"Change alone is the single most important metric when it comes to quality issues in code."* — Tornhill

The idea: your version-control history is data. Mining it tells you where
bugs hide, which files implicitly couple, who owns knowledge, and where
to spend a refactor budget. Behavioral analysis adds the **temporal** axis
that static analysers lack.

Key techniques:
- **Hotspots** — complex code × frequent change. Highest ROI for refactoring.
- **Change coupling / temporal coupling** — files that change in the same
  commit again and again, even though the code seems unrelated.
- **Knowledge maps / author churn** — who knows what, and what's at risk
  if they leave.
- **Code health trends** — track health over time, not as a snapshot.

Tools:
- **[code-maat](https://github.com/adamtornhill/code-maat)** — open-source CLI, Clojure, mines `git`/`hg`/`svn`/`p4`/`tfs` logs.
- **[CodeScene](https://codescene.com)** — commercial successor by Tornhill's Empear. Adds ML, code health, vulnerability prediction. Free community tier on GitHub.
- **[code-forensics](https://github.com/smontanari/code-forensics)** — OSS toolkit that combines code-maat with visualisations.

Primary sources:
- [Software Design X-Rays — Pragmatic](https://pragprog.com/titles/atevol/software-design-x-rays/)
- [Adam Tornhill's homepage](https://www.adamtornhill.com/)
- [Hotspots in CodeScene docs](https://docs.enterprise.codescene.io/versions/3.0.2/guides/technical/hotspots.html)

---

## Family B — Working Effectively With Legacy Code (Michael Feathers)

> *"Legacy code is simply code without tests."* — Feathers

Two ideas you can't unsee after reading WELC:

1. **Seams** — a place where you can change program behavior without
   editing in that place. Types: preprocessor, link, object. Object seams
   are usually preferable in OO languages.
2. **Characterization tests** — tests that pin down what the code
   *actually* does (not what spec says). You probe the code with inputs,
   capture outputs, and assert them. The test becomes a safety net while
   you refactor.

The book is essentially an algorithm catalogue: Sprout Method, Sprout
Class, Wrap Method, Wrap Class, Extract and Override Call, Subclass and
Override Method, "I can't get this class into a test harness" pattern,
"I can't run this method in a test harness" pattern, etc.

Primary sources:
- [WELC summary — understandlegacycode.com](https://understandlegacycode.com/blog/key-points-of-working-effectively-with-legacy-code/)
- [InfoQ podcast — Feathers looking back at WELC](https://www.infoq.com/podcasts/working-effectively-legacy-code/)

---

## Family C — Approval / Golden-Master Testing

Mechanised characterization tests. The library captures output to an
"approved" file; subsequent runs diff against it. Workflow:
**Arrange → Act → Print → Compare**.

Tools:
- **[ApprovalTests](https://approvaltests.com)** — Llewellyn Falco's polyglot library: .NET, Java, JS/TS, PHP, Python, Ruby, C++, Go, Perl, Obj-C.
- **TextTest** (Emily Bache)
- **[Verify](https://github.com/VerifyTests/Verify)** (Simon Cropp) — modern .NET / TS alternative.
- **jest --snapshot**, **insta** (Rust), **syrupy** (Python) — snapshot testing in the wild.

People to follow:
- [Emily Bache](https://github.com/emilybache) — refactoring katas (especially [GildedRose](https://github.com/emilybache/GildedRose-Refactoring-Kata) in 40+ languages), [Coding Is Like Cooking](https://coding-is-like-cooking.info/) blog, *The Coding Dojo Handbook*.
- [Llewellyn Falco](https://approvaltests.com/) — talks, Strong-style pairing, Mob Programming co-developer.

When to reach for it: a legacy function with no spec, where you don't even
know what it should do — let alone whether it's right.

---

## Family D — Refactoring catalogs & methods

Books:
- **[Refactoring, 2nd ed](https://martinfowler.com/books/refactoring.html)** — Martin Fowler. ~70 refactorings, JS examples, free web-edition extras.
- **Refactoring to Patterns** — Joshua Kerievsky.
- **Five Lines of Code** — Christian Clausen (more recent, opinionated rules).

Methods / patterns:
- **[Mikado Method](https://www.manning.com/books/the-mikado-method)** — Ellnestam & Brolund. Try the change; when it breaks, *revert*, write down the prerequisite, and recurse. The result is a DAG of "what I need first". Execute from the leaves.
- **[Strangler Fig pattern](https://martinfowler.com/bliki/StranglerFigApplication.html)** — Fowler, 2004. Wrap the old system in a façade and divert traffic to new services piece by piece until the old is dead. [Shopify post](https://shopify.engineering/refactoring-legacy-code-strangler-fig-pattern) is a great real-world write-up.
- **Branch by Abstraction** (Paul Hammant) — a sibling pattern for in-process replacement.
- **Parallel Change / expand-contract** (Danilo Sato).

---

## Family E — Static-analysis & code-quality platforms

| Tool | Languages | Notable feature |
|---|---|---|
| [SonarQube](https://www.sonarsource.com/products/sonarqube/) | 30+ | Cognitive complexity, tech-debt minutes, A–E rating, 6000+ rules |
| Code Climate | Multi | Hosted, churn ↔ complexity overlay |
| [NDepend](https://www.ndepend.com/) | .NET | CQLinq queries, dependency matrix |
| [Structure101](https://structure101.com/) | Java / .NET / C++ | DSM, cycle-breaker, layering |
| [Lattix](https://www.lattix.com/) | Java / .NET / C/C++ / Ada | DSM edit-in-place |
| [SciTools Understand](https://scitools.com/) | 20+ incl. safety-critical | MISRA/DO-178C, onboarding graphs |
| [CAST Highlight](https://www.castsoftware.com/products/highlight) | 30+ | Portfolio-scale, cloud-readiness, M&A |
| [SIG Sigrid](https://www.softwareimprovementgroup.com/sigrid-software-excellence-platform/) | Multi | Benchmarked against 25 000 industry systems |
| Fortify / Checkmarx / Veracode | Multi | Security-leaning |

For most teams: start with SonarQube (free), add a behavioral layer
(CodeScene community) when you need to prioritise across a large repo.

---

## Family F — Architecture-as-tests (Fitness Functions)

Encode architectural rules as code that fails CI when broken. The "evolutionary
architectures" idea from Neal Ford, Rebecca Parsons, Patrick Kua.

- **[ArchUnit](https://www.archunit.org/)** (Java/Kotlin) — fluent rules in
  JUnit. Layering, cycles, naming, can ingest PlantUML as the spec.
- **[Deptrac](https://deptrac.github.io/deptrac/)** (PHP) — YAML layer config, CI exit code, Mermaid/Graphviz output.
- **[jdeps](https://docs.oracle.com/en/java/javase/11/tools/jdeps.html)** — JDK-bundled class/JAR dependency tool, `-dotoutput`.
- **NetArchTest** (.NET), **Konsist** (Kotlin), **Fitness** (Python), **good-fences** (TS).

Use them to prevent the slow drift that turns a clean codebase into the
next legacy system.

---

## Family G — Structural search & large-scale refactoring (codemods)

Tools that let you rewrite code mechanically across thousands of files —
either through ASTs or structural templates.

- **[ast-grep](https://ast-grep.github.io/)** — Rust, tree-sitter, polyglot, fast.
- **[Semgrep](https://semgrep.dev/)** — AST patterns, huge rule library, security focus.
- **[comby](https://comby.dev/)** — language-agnostic structural template ("holes"), not AST-aware.
- **[jscodeshift](https://github.com/facebook/jscodeshift)** — JS/TS AST codemods (Facebook).
- **ts-morph** — programmatic TS AST.
- **[OpenRewrite](https://docs.openrewrite.org/)** — Lossless Semantic Trees for Java/Kotlin/Python/JS; huge recipe catalog; **Moderne** is the commercial enterprise platform on top.

Trend: codemods + LLM. Semgrep's AutoFixes route through an LLM; Moderne
runs OpenRewrite recipes across thousands of repos simultaneously.

Reading: [Martin Fowler — Refactoring with Codemods to Automate API Changes](https://martinfowler.com/articles/codemods-api-refactoring.html).

---

## Family H — Semantic & variant analysis (code as a database)

- **[CodeQL](https://codeql.github.com/)** — GitHub's Datalog-flavoured query language. Extracts an AST + name/type bindings into a relational DB, runs queries over it. Famous for **variant analysis**: take a known CVE and find every analogue across 1000+ repos (MRVA in VS Code).
- **Glean** (Meta) — fact-based code knowledge graph; powers Meta's IDEs.
- **Stack Graphs** (GitHub) — incremental, multi-language name resolution.
- **[LSIF](https://lsif.dev/)** / **SCIP** — index formats consumed by Sourcegraph and others to provide cross-repo Go-To-Definition.

---

## Family I — Visualisation: maps, cities, animations

- **[CodeCity](https://wettel.github.io/codecity.html)** — research origin of the metaphor. Classes = buildings, packages = districts.
- **[CodeCharta](https://codecharta.com/)** (MaibornWolff, OSS) — production-ready 3D treemap city. Imports from SonarQube, Tokei, code-maat. Supports delta comparison and 3D printing.
- **[Gource](https://gource.io/)** — mesmerising animation of repo history.
- **[git-of-theseus](https://github.com/erikbern/git-of-theseus)** (Erik Bernhardsson) — stacked-area "how much code from year X still lives" plot.
- **CodeViz** (VS Code), **Emerge**, **repostats**, **Lucidchart Code Map**.

---

## Family J — Onboarding, navigation, knowledge-sharing

- **[Sourcetrail](https://github.com/CoatiSoftware/Sourcetrail)** — once-popular OSS interactive code map; project was discontinued in 2021 but the binary still works for C/C++/Java/Python.
- **[CodeSee](https://www.codesee.io/)** — auto-generated maps + "tours" you can record so newcomers replay how a change flows through the system.
- **[Swimm](https://swimm.io/)** — docs that live next to code and auto-update when the referenced code changes.
- **[Sourcegraph](https://sourcegraph.com/)** — universal code search, cross-repo Go-To-Def, batch changes, Code Insights trends.
- **AI / agentic layer** — [Sourcegraph Cody](https://sourcegraph.com/cody) (RAG over your repos), [Cursor](https://cursor.com), [Aider](https://aider.chat) (tree-sitter repo map), [Cline](https://cline.bot), Claude Code. These finally make "ask the codebase a natural-language question" reliable.

---

## Family K — Moldable Development (feenk / Glamorous Toolkit)

Tudor Gîrba and feenk argue that the way we read code today — a wall of
text in a text editor — is the actual bottleneck. The answer is **moldable
development**: build many small, throwaway, contextual tools per system,
so the *environment* answers your questions instead of your brain.

Components:
- **[Glamorous Toolkit (GT)](https://gtoolkit.com)** — Pharo-based moldable IDE. v1.0 in 2022 after 6 years of dev / 14 years of research. MIT.
- **GT Inspector** — every object has multiple custom *views* tailored to that object. Open a collection and you see "Items / Sum / SQL / Map".
- **GT Documenter / Lepiter** — live notebooks where prose, executable snippets, and visualisations all live together.
- **GT Coder** — expandable editors, "examples" double as tests and as docs.
- **Bloc** — graphics layer that makes all of this possible.
- **[Moose](https://moosetechnology.org/)** — older Pharo platform, FAMIX language-independent metamodel, used by [Synectique](https://www.pharo.org/success/Synectique) for legacy audits. CodeScene's intellectual sibling on the research side.

Reading: [book.gtoolkit.com](https://book.gtoolkit.com), [Tudor Gîrba's talks](https://www.youtube.com/@feenkcom).

---

## Family L — Mutation testing (the test for your tests)

Mutate the production code (flip `>` to `>=`, delete a return, etc.); rerun
the test suite. A surviving mutant means your tests can't tell.

- **[PIT / PITest](https://pitest.org/)** — Java, most mature. Maven/Gradle/IDE.
- **[Stryker](https://stryker-mutator.io/)** — JS/TS, C#, Scala.
- **mutmut** / **cosmic-ray** — Python.
- **mull** — LLVM, native code.
- **[Microsoft's built-in .NET mutation testing](https://learn.microsoft.com/en-us/dotnet/core/testing/mutation-testing)**.

Mutation score is a far more honest metric than line coverage.

---

## Family M — Property-based / generative testing

Don't write examples — write *invariants* and let the framework generate
adversarial inputs.

- **[QuickCheck](https://hackage.haskell.org/package/QuickCheck)** — Haskell, original.
- **[Hypothesis](https://hypothesis.works/)** — Python, widest industry adoption, supports stateful tests.
- **fast-check** (JS/TS), **jqwik** (Java), **Hedgehog** (Haskell/F#/Scala), **PropEr** (Erlang).

For legacy code: powerful for round-trip / idempotency / monotonicity
properties. ["Property Tests for Legacy Code"](https://matt.diephouse.com/2020/02/property-tests-for-legacy-code/) is a good
introduction.

---

## Family N — Architecture documentation & decision capture

- **[C4 model](https://c4model.com/)** (Simon Brown, 2006-2011) — four levels: Context, Containers, Components, Code.
- **[Structurizr](https://structurizr.com/)** — Simon Brown's DSL/tooling. One model → PlantUML, Mermaid, draw.io, interactive diagrams.
- **[Architecture Decision Records (ADRs)](https://adr.github.io/)** — Michael Nygard. One short markdown per decision. Toolchain: `adr-tools` (Nat Pryce), Log4brains, MADR template. [Fowler bliki entry](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html).
- **[Living Documentation](https://www.amazon.com/Living-Documentation-Cyrille-Martraire/dp/0134689321)** — Cyrille Martraire. Make documentation a side-effect of code: generate from tests/types/annotations.
- **[Event Storming](https://www.eventstorming.com/)** (Alberto Brandolini) — big-room sticky-note workshop to recover business flow. Pairs well with legacy modernisation.
- **[Domain Storytelling](https://domainstorytelling.org/)** — Hofer & Schwentner. Smaller, more moderated; the actor does X with Y.

---

## Family O — Runtime / dynamic understanding

Static analysis tells you what *can* happen. Tracing tells you what *did*.

- **[Flame Graphs](https://www.brendangregg.com/flamegraphs.html)** — Brendan Gregg. Sampled stack visualisation; the workhorse for "where is time actually spent". Off-CPU flame graphs cover I/O too.
- **eBPF tooling** (`bcc`, `bpftrace`) — production-safe kernel tracing.
- **OpenTelemetry** + Jaeger/Tempo/DataDog/Honeycomb — service maps and distributed traces for microservices.
- **Reverse-engineering profilers** — py-spy, async-profiler, perf, dtrace.
- **[rr](https://rr-project.org/)** (Mozilla) — record-and-replay debugger, deterministic time-travel.
- **[Pyroscope](https://pyroscope.io/)** — continuous profiling.

---

## Family P — Nicolas Carlo's *Legacy Code: First Aid Kit*

[understandlegacycode.com](https://understandlegacycode.com/) is the most
day-to-day-practical blog in the field. His e-book bundles 14 techniques
into a workflow you can apply on a Monday morning. The canonical 14 (titles
paraphrased):

1. Architecture Decision Records (ADRs)
2. **Brain Dump** — write out everything you currently believe about a
   piece of code; this reveals your knowledge gaps faster than reading.
3. Over-commit (use cheap commits as scratch space).
4. Mikado Method.
5. Hotspots Analysis.
6. Approval Testing.
7. Coding Katas (sharpen on safe ground).
8. Characterization tests.
9. Working with seams.
10. Strangler Fig.
11. Documenting the *why*.
12. Refactor-then-feature ordering.
13. (Static-typing-only) Move-fast-with-types techniques.
14. (Static-typing-only) Compiler-driven development.

Three talks he frequently recommends:
- **["All the Little Things"](https://www.youtube.com/watch?v=8bZh5LMaSmE)** — Sandi Metz.
- **["Therapeutic Refactoring"](https://www.youtube.com/watch?v=J4dlF0kcThQ)** — Katrina Owen.
- **["Approval Testing"](https://www.youtube.com/results?search_query=approval+testing+llewellyn+falco)** — Llewellyn Falco.

---

## A practical "starter recipe"

If you inherited a 500-kLoC codebase tomorrow, this is the order I'd run
in:

1. **Mine VCS** → run `code-maat` or CodeScene Community. Print the top-20
   hotspots and the top-20 temporal-coupling pairs.
2. **Map structure** → SciTools Understand / NDepend / Structure101, or
   write ArchUnit / Deptrac rules to assert your current invariants.
3. **Visualise for the team** → CodeCharta city plus a one-page C4 Context
   + Container diagram in Structurizr.
4. **Pick the top hotspot.** Don't refactor yet.
5. **Cover the hotspot with approval / characterization tests.** Capture
   the current behavior as the reference.
6. **Mikado-plan** the refactor. Sprout/Wrap new behavior, never touch
   what isn't already pinned by a test.
7. **Mutation-test** the new tests (PIT / Stryker) so you trust them.
8. **Lock in invariants** with new ArchUnit/Deptrac rules so the issue
   can't come back.
9. **Persist understanding** via ADRs and Living-Documentation artefacts;
   record a CodeSee tour or a GT notebook for the next teammate.
10. **Strangler-Fig** out toward the new architecture, one slice at a
    time. Repeat from step 1 quarterly.

---

## Quick links — primary websites

- [understandlegacycode.com](https://understandlegacycode.com) — Nicolas Carlo
- [adamtornhill.com](https://www.adamtornhill.com) — Adam Tornhill
- [codescene.com](https://codescene.com)
- [code-maat on GitHub](https://github.com/adamtornhill/code-maat)
- [feenk.com](https://feenk.com) / [gtoolkit.com](https://gtoolkit.com) / [book.gtoolkit.com](https://book.gtoolkit.com)
- [moosetechnology.org](https://moosetechnology.org)
- [approvaltests.com](https://approvaltests.com)
- [coding-is-like-cooking.info](https://coding-is-like-cooking.info) — Emily Bache
- [refactoring.com](https://refactoring.com) — Martin Fowler
- [martinfowler.com/bliki/](https://martinfowler.com/bliki/) — bliki entries
- [c4model.com](https://c4model.com) / [structurizr.com](https://structurizr.com)
- [adr.github.io](https://adr.github.io)
- [archunit.org](https://www.archunit.org)
- [pitest.org](https://pitest.org) / [stryker-mutator.io](https://stryker-mutator.io)
- [semgrep.dev](https://semgrep.dev) / [ast-grep.github.io](https://ast-grep.github.io) / [comby.dev](https://comby.dev)
- [codeql.github.com](https://codeql.github.com)
- [sourcegraph.com](https://sourcegraph.com)
- [brendangregg.com/flamegraphs.html](https://www.brendangregg.com/flamegraphs.html)
- [eventstorming.com](https://www.eventstorming.com) / [domainstorytelling.org](https://domainstorytelling.org)
