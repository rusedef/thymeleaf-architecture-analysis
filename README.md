# Thymeleaf — Software Architecture Analysis

> **Course:** BBM485 Software Architectures · Hacettepe University, Department of Computer Engineering  
> **Supervisor:** Prof. Ayça Kolukısa  
> **Year:** 2026  
> **Subject:** [Thymeleaf](https://www.thymeleaf.org/) — Open-source Java server-side template engine

This repository contains a two-phase architectural analysis of the Thymeleaf open-source project (110 KLOC · 1,901 classes). The study evaluates the project's **concrete architecture** through software metrics and dependency analysis, and its **conceptual architecture** through architectural views, pattern identification, and code quality assessment.

---

## Documents

| Document | Description |
|---|---|
| [`T1_Report.pdf`](./Thymeleaf_SoftwareArchitecture_T1_Report.pdf) | Concrete Architecture — metrics analysis, dependency matrix, dependency graphs |
| [`T1_Appendix-A.pdf`](./T1_Appendix-A_gqm_tree_thymeleaf.pdf) | GQM (Goal-Question-Metric) tree |
| [`T1_Appendix-B.pdf`](./T1_Appendix-B_dependency_matrix.pdf) | Full dependency matrix |
| [`T1_Appendix-C.pdf`](./T1_Appendix-C_dependency_graphs.pdf) | 20 package-level dependency graphs (L3 + L4) |
| [`T2_Report.pdf`](./Thymeleaf_SoftwareArchitecture_T2_Report.pdf) | Conceptual Architecture — architectural views, pattern identification, SonarQube analysis |
| [`T2_Appendix-A.pdf`](./T2_Appendix-A.pdf) | Context diagram |
| [`T2_Appendix-B.pdf`](./T2_Appendix-B.pdf) | Component & Connector view diagrams |

---

## Phase 1 — Concrete Architecture (T1 Report)

### Metrics Analyzed (26 Total)

| Category | Metrics |
|---|---|
| C&K Metrics | CBO, RFC, WMC, LCOM, DIT, NOC |
| MOOD Metrics | CF, AIF, MIF, AHF, MHF, PF |
| Halstead Metrics | H_V, H_D, H_E, H_B, H_N, H_n, H_N1, H_N2, H_n1, H_n2 |
| Other | CC (Cyclomatic Complexity), NNL, NOM, MI, LOC, SLOC |

**Tools:** SciTool Understand · CKJM Extended · javalang (Halstead)

### Key Findings

- **Maintainability Index:** MI = 73.27 (median) — classified as "Good" maintainability band
- **Coupling Between Objects:** CBO = 7 (median) — within acceptable range
- **Coupling Factor:** CF = 21% — indicates moderate inter-class coupling
- **Cyclomatic Complexity:** CC = 12.33 (median) — slightly elevated; flags refactoring candidates

### Circular Dependency Analysis

Dependency matrix and 20 package-level dependency graphs (L3 + L4) were produced. A critical finding was identified in the `engine` package — the most heavily depended-upon package in the system:

| Cycle | Outgoing | Incoming |
|---|---|---|
| engine ↔ model | 485 | 7 |
| engine ↔ util | 186 | 35 |
| engine ↔ context | 34 | 108 |
| engine ↔ cache | 16 | 42 |

> **Note:** The dependency matrix alone was insufficient to detect these cycles. Cross-referencing with the dependency graphs corrected a misinterpretation for the `util` and `exceptions` packages.

---

## Phase 2 — Conceptual Architecture (T2 Report)

### Architectural Views Produced

**Module View — 5-Layer UML Package Diagram**

```
Layer 1 — API             org.thymeleaf root (TemplateEngine, ITemplateEngine)
Layer 2 — Core            engine (88 files) · standard (144 files)
Layer 3 — Template        templateparser · templateresolver · templateresource · templatemode
Layer 4 — Processing      context · model · expression · dialect · inline · processor
                          preprocessor · postprocessor · linkbuilder · messageresolver
Layer 5 — Infrastructure  cache · util · exceptions · web
```

**Component & Connector (C&C) View**

6 components: TemplateEngine · StandardDialect · StandardCacheManager · TemplateResolver · Message & Link · Host Application

**Allocation View**

Deployment diagram covering 3 nodes: JVM Application Server · File System · Client Machine

### Architectural Patterns Identified

- **Layered Architecture** — strict dependency direction from top (API) to bottom (Infrastructure)
- **Pipes and Filters** — template processing pipeline: `preprocessor → processor → postprocessor`

---

## SonarQube Code Quality Analysis

**Total findings: 3,819**

| Severity | Count |
|---|---|
| BLOCKER | 8 |
| CRITICAL | 306 |
| MAJOR | 1,224 |
| MINOR | 1,421 |
| INFO | 860 |

### 8 Blocker Findings — Detailed Analysis

#### Blocker-1 — Empty Test Class `(java:S2187)`
**File:** `tests/.../engine/CloseElementTagTest.java:31`

The `CloseElementTagTest` class is declared as a JUnit test class but contains no test methods. The test runner reports this class as passing — meaning a component appears tested when it has not been tested at all.

**Fix:** Add at least one `@Test` method covering `CloseElementTag` functionality, or annotate the class with `@Disabled` if implementation is deferred.

---

#### Blocker-2 — Bitwise `&` Used as Logical Operator `(java:S2178)`
**File:** `tests/.../templateparser/text/TextTraceEvent.java:103`

The `&` operator is used in place of `&&` in a boolean expression. Unlike `&&`, the `&` operator always evaluates both operands — if the right-hand side contains a null reference when the left-hand side is false, a `NullPointerException` may be thrown at runtime.

**Fix:** Replace `&` with `&&` on the flagged line. A 5-minute fix that eliminates a potential runtime failure.

---

#### Blocker-3 — Field Shadowing `(java:S2387)` — 2 Occurrences
**Files:**
- `lib/thymeleaf-spring6/.../ThymeleafReactiveView.java:105`
- `lib/thymeleaf-spring5/.../ThymeleafReactiveView.java:105`

Both `ThymeleafReactiveView` subclasses redeclare a field named `logger` that already exists in the parent class `AbstractView`. Subclass methods use the subclass logger while inherited methods use the parent's logger — making it ambiguous which class a log record originates from. The identical defect in both Spring 5 and Spring 6 modules indicates propagation through copy-paste.

**Fix:** Remove the `logger` field from both subclasses and use the inherited field directly. If the parent's field is `private`, update its access modifier to `protected`.

---

#### Blocker-4 — Defective `clone()` Implementation `(java:S2975)` — 3 Occurrences
**Files:**
- `lib/thymeleaf/.../StandardJavaScriptSerializer.java:489`
- `lib/thymeleaf/.../OGNLExpressionObjectsWrapper.java:138`
- `lib/thymeleaf/.../CharArrayWrapperSequence.java:110`

All three classes use Java's `Cloneable` interface and `Object.clone()`. This mechanism is brittle: mutable fields are shallow-copied, leading to shared references and unexpected side effects. Notably, `CharArrayWrapperSequence` belongs to the `util` package — consumed by virtually the entire system, as confirmed by the T1 dependency analysis — making its blast radius significant.

**Fix:** Remove `clone()` from all three classes and replace with a copy constructor or a static factory method (`of(source)`) providing equivalent deep-copy behavior. Remove `implements Cloneable` from all three classes.

---

#### Blocker-5 — Field Name Clash: `empty` / `EMPTY` `(java:S1845)`
**File:** `lib/thymeleaf/.../engine/IteratedGatheringModelProcessable.java:715`

The class declares both a static constant `EMPTY` and an instance variable `empty`. While Java is case-sensitive, the near-identical names create serious confusion. Mistakenly mixing them up produces silent logic errors. This class resides within the `engine` package — the package with the highest dependency count in T1 (485 outgoing), meaning any error here has wide systemic impact.

**Fix:** Rename the `empty` instance variable to a descriptive name such as `isCurrentIterationEmpty` and propagate the rename consistently across all methods in the class.

---

## Improvement Recommendations

### 1. Break Circular Dependencies (Dependency Inversion Principle)

The four bidirectional cycles involving the `engine` package should be resolved by introducing abstractions:

- `IEngineContext` interface — to decouple `engine ↔ context`
- `ITemplateModel` interface — to decouple `engine ↔ model`
- `ICacheAccess` interface — to decouple `engine ↔ cache`
- Utility functions consumed by `engine` moved to a sub-package — to decouple `engine ↔ util`

### 2. Reduce Cognitive Complexity (Extract Method Refactoring)

Methods flagged for high cognitive complexity (CC > 15) should be refactored using the Extract Method pattern — breaking complex logic into smaller, single-purpose private methods. This directly improves the Maintainability Index.

---

*Developed by [Rukiye Sedef Keskin](https://github.com/rusedef)*
