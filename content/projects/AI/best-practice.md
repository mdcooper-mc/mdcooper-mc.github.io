+++
title = "Best Practice"
description = "A practical engineering handbook covering design principles, patterns, testing discipline, AI/RAG architecture, agent behaviour, and documentation standards."
+++

---

## General Engineering Principles

### The Foundation: Four Principles Every Codebase Needs

Good software starts with four principles that work together. The **Single Responsibility Principle (SRP)** says that every module, class, and function should have exactly one reason to change. When a unit handles multiple concerns, a change in one area risks breaking another, and tests become difficult to reason about because behaviour spans unrelated paths. Keeping things focused makes refactoring safe and failures easy to isolate.

**Don't Repeat Yourself (DRY)** means every piece of domain knowledge has one unambiguous home in the codebase. Duplication is not just a code smell — it is a maintenance liability. When the same rule exists in three places, they will eventually diverge. Consolidate logic into shared abstractions and your system stays consistent as it grows.

**Keep It Simple** is the hardest principle to follow under pressure. Complexity compounds faster than most engineers expect. Every unnecessary abstraction, every premature optimisation, and every clever trick adds cognitive load for the next person. Prefer the simplest solution that solves the actual problem, and resist the temptation to build for imagined futures.

**Interface-Driven Design** ties the others together. All code should depend on abstractions, not concrete implementations. Use interfaces as the type for variables, fields, parameters, and return values everywhere. Concrete types belong only at composition roots. This gives you loose coupling, easy substitution, clean mocking in tests, and enforced boundaries between domains.

---

### Dependency Injection and Architecture Layers

Dependency injection is the mechanism that makes interface-driven design real. Every dependency must be provided externally rather than created internally. Use constructor injection to make dependencies explicit and visible. Keep constructors small — large constructor signatures indicate misplaced responsibilities. Never pass the container or registry through the system; that is a service locator, not injection.

A DI-driven architecture has four layers. The **domain layer** contains entities, value objects, domain services, and domain interfaces; it has no external dependencies. The **application layer** contains use cases, orchestrators, and workflows; it depends only on domain interfaces. The **infrastructure layer** contains concrete implementations of repositories, components, and integrations. The **composition root** assembles the entire application and wires interfaces to their implementations; it is the only place concrete types should appear.

Common DI anti-patterns to avoid: circular dependencies, constructors that perform I/O or heavy work, injecting concrete classes instead of abstractions, hidden dependencies, static singletons, and service locator patterns.

---

### Interface Design and Builder-Flow Patterns

Not every class needs an interface. Introduce interfaces only when abstraction provides real value — when behaviour is domain-facing, when there are multiple implementations, or when the caller must be isolated from the infrastructure. Interfaces must be behaviour-based, not data-based, and should represent a domain capability rather than mirror a single class. Prefer capability-based naming such as `UserRepository`, `EventPublisher`, or `ModelProvider`.

**Builder-flow interfaces** eliminate temporal coupling by enforcing valid call order at compile time. Each interface represents a stable state in the workflow, not an implementation detail. Each method must return a different interface — the next stage — and each stage exposes only the operations valid at that point. Stage interfaces must be minimal, behaviour-based, and composable. Methods must never return concrete classes; only stage interfaces. Stage transitions must be deterministic: the same input always produces the same next interface.

Functional interface patterns reduce branching and cyclomatic complexity by representing a single unit of behaviour: handlers that process input to output, mappers that transform values, predicates that evaluate conditions, and validators that check domain constraints.

---

### Design Patterns

Patterns are vocabulary for solutions that recur across different problems. **Creational patterns** control how objects come into existence: Factory Method, Builder, Abstract Factory, Prototype, and Singleton. **Structural patterns** describe how components are assembled: Adapter, Decorator, Facade, Composite, Proxy, Bridge, and Flyweight. **Behavioral patterns** define how objects communicate: Strategy, Observer, Command, Chain of Responsibility, State, Template Method, Visitor, Mediator, and Memento.

Beyond the Gang of Four, architectural patterns are now standard. **Event Sourcing** stores state as an immutable sequence of events. **CQRS** separates read and write models. The **Repository pattern** abstracts data access behind a domain-facing interface. The **Service Layer** encapsulates business operations as explicit, reusable services. The **Saga pattern** coordinates distributed transactions. **Null Object** eliminates null checks with a do-nothing implementation.

For control flow reduction, prefer: Strategy and Polymorphic Dispatch over conditionals, Null Object and Option/Maybe over null checks, State machines over state-based branching, Pipeline Pattern for linear multi-stage flows, Either/Result for error paths without exception-based control flow, and Lookup Tables for data-driven selection.

---

### Functional Programming

Functional programming is not an all-or-nothing paradigm — most of its value comes from applying its principles selectively inside any language or architecture.

The core idea is **immutability**: data does not change once created. New values are produced from old ones through transformation. This eliminates a wide class of bugs caused by shared mutable state and simplifies concurrency because there is nothing to lock.

**Pure functions** take inputs and return outputs with no side effects and no hidden dependencies. Given the same inputs, they always produce the same output. Pure functions are trivially testable, safe to cache, and safe to run in parallel.

**Higher-order functions** enable composition: building complex behaviour by chaining small, focused transformations. For error handling, **railway-oriented programming** models success and failure as two tracks through a pipeline using an Either type. The **Functional Core / Imperative Shell** architecture keeps all business logic in pure functions and pushes side effects to the outermost shell.

---

### Object-Oriented Design

OOP is most effective when it mirrors the domain. **Entities** have identity that persists over time. **Value Objects** are defined entirely by their attributes and carry no identity; treat them as immutable. **Aggregates** cluster related entities and value objects into a consistency boundary with the aggregate root as the only entry point.

Favour composition over inheritance: assemble behaviour by combining small, focused interfaces rather than building deep class hierarchies. Hierarchies are rigid; composition is flexible. OOP and functional programming are not opposites — use pure functions for deterministic data transformations and objects with interfaces for stateful or polymorphic behaviour.

---

### API-First and REST Principles

Design APIs before implementing services or UI. API contracts define behaviour, not code, and must never break existing clients — evolve via versioning. The same input must always yield the same output (deterministic APIs), and operations must be safe to retry without side effects (idempotent APIs).

REST resources are modelled as nouns with a stable URI, operated on via standard methods (GET, POST, PUT, PATCH, DELETE). Responses must be stateless, cacheable, and self-describing. HATEOAS drives workflow through server-provided links rather than hard-coded routes in clients.

---

### Testing Discipline

Every test must follow **Arrange–Act–Assert**. Arrange sets up the context. Act executes exactly one behaviour. Assert verifies the outcome with specific, minimal expectations. Each test method covers one use case only — no branching, no multiple scenarios. When a function contains logic irrelevant to the scenario being verified, use a mock that throws a controlled exception at the first opportunity to fail fast and avoid building deep mock chains.

Organise tests by domain, not by technical layer. The test directory mirrors the source directory exactly. Aim for 70–90% coverage on critical paths, edge cases, and error conditions. Mock only external systems and expensive operations; avoid mocking internal logic.

---

### Code Size, Complexity, and Package Structure

Files beyond 200–500 lines usually contain mixed responsibilities. Classes with more than ten methods are probably doing too much. Functions longer than twenty lines almost certainly have a simpler decomposition. Cyclomatic complexity — the number of independent execution paths through a function — should be kept low. Deep nesting beyond two or three levels signals the need for restructuring with early returns, extracted helpers, or a more appropriate pattern.

Organise code by **domain**, not by technical role. A folder called `payments/` tells you more than `services/`. Each domain package contains everything relevant to that domain: models, interfaces, implementations, factories, validators, and tests. Expose only what needs to be public. File names must not repeat the package name — the directory path already provides that context. Use absolute imports at the top of files only; avoid circular imports.

---

### Architecture and Reliability

Design for **observability** from the start: structured logging, metrics, and distributed tracing are not optional extras. **Graceful degradation** means the system continues to provide partial value when a dependency fails. Fail fast on invalid input at the boundary; never let bad data propagate deeper. Externalize configuration and validate it at startup.

**Infrastructure as Code** means environments are defined, versioned, and reproduced consistently. Quality gates — linting, type checking, coverage thresholds, security scanning — belong in CI. Blue-green and canary deployments shift traffic gradually to new versions.

---

## AI and RAG Best Practices

### Machine Capabilities and Configuration

Always read `reference/documentation/machine-capabilities.md` before selecting models, precision, batch sizes, or device placement. Extract GPU count, VRAM, compute capability, CPU cores, RAM, disk type, and network bandwidth. Never hard-code machine assumptions; always derive from the machine-spec file. Configure default precision, max batch size per model, max concurrent inference requests, and device placement rules.

---

### Core Architecture Components

Well-structured AI systems are composed of isolated, interface-backed services. The **Model Loader Service** handles lazy loading and model lifecycle. The **Inference Service** performs forward passes with batching and precision control. The **Embedding Service** produces vector embeddings for retrieval. The **Vector Store Service** manages approximate nearest-neighbour indexes. The **RAG Orchestrator** coordinates retrieval, ranking, context building, and generation. The **Preprocessing Pipeline** handles tokenisation, normalisation, and chunking. The **Postprocessing Pipeline** handles output cleaning, validation, and structuring.

Define interfaces for `ModelProvider`, `EmbeddingService`, `VectorStore`, and `RAGService`. Core logic must depend only on these interfaces, not concrete implementations. Enforce SRP: model loading is not inference, retrieval is not ranking, preprocessing is not postprocessing, and logging is not business logic.

---

### Model Loading, Batching, and Lifecycle

Load models only when first needed and keep them in memory while active. Unload when idle or under memory pressure. Maintain a model registry with loader functions, reference counts, and device placement rules. Never load a model per request.

Batch requests by model type and task type. Run a single forward pass per batch and split outputs back to individual responses. Batch requests to the same model together even across tasks to minimise load/unload cycles and maximise GPU utilisation.

Use `torch.no_grad()` for inference. Use `torch.cuda.amp.autocast()` for mixed precision. Avoid repeated `.to(device)` calls. Use `torch.compile(mode="reduce-overhead")` for inference graph optimisation. Pin model versions for reproducibility.

---

### Precision, Quantisation, and Compilation

Use FP16 or BF16 for GPU inference. Use INT8 or 4-bit quantisation (QLoRA/GPTQ) for large models on limited VRAM. Validate accuracy impact before deploying quantised models.

Export models to ONNX for CPU optimisation, GPU acceleration, and cross-platform inference. Use ONNX optimisers (constant folding, graph pruning, operator fusion) and execution providers (CUDA EP, TensorRT EP, OpenVINO EP). Use TensorRT for FP16/INT8 acceleration and kernel fusion for maximum GPU throughput.

---

### RAG Architecture and Token Optimisation

The RAG flow is: query preprocessing → query embedding → vector retrieval → reranking → context window construction → generation → postprocessing. Use semantic chunking for long documents. Use rerankers for higher accuracy when latency allows. Cache embeddings and retrieval results. Evaluate with recall@k, MRR, and semantic similarity metrics.

Minimise prompt length. Use summarised context where possible. Use token-efficient formats such as JSON or bullet lists. Use prompt compression models for large contexts. Track token budget to avoid exceeding model context limits.

---

### Parameter Initialisation

Never initialise weights or biases to zero. Zero initialisation causes symmetry problems where all neurons learn identical features, flat gradients producing a misleading hockey-stick loss curve, and delayed convergence that optimisers cannot break. Use Kaiming/He initialisation for ReLU/GELU activations, Xavier/Glorot for tanh/sigmoid, and orthogonal initialisation for RNNs and deep linear layers. Initialise biases to small non-zero constants to prevent dead neurons in early layers.

When evaluating models, plot metrics on a log scale to detect subtle early changes and use moving averages to smooth noise without hiding instability.

---

### Testing and Observability

Unit test preprocessing and postprocessing as pure functions and RAG orchestration with mocks. Run integration tests with small end-to-end indexes. Run performance tests for latency per stage and throughput under load. Track regressions using golden datasets when changing models or prompts.

Log model name, latency, device, and batch size. Track GPU and CPU utilisation. Use distributed tracing for multi-service RAG. Validate outputs for safety and domain correctness. Never log raw PII or full prompts.

---

## Agent Practices

### General Reasoning

Always understand the full context before raising concerns; do not assume something is wrong without clear evidence. Consider that unusual patterns may be intentional, domain-specific, or required by external constraints. Think through the broader system: behaviour may depend on environment, calling conventions, or upstream and downstream components. Do not treat unfamiliar or unconventional patterns as errors — investigate usage and intent before drawing conclusions. Prefer explanation over accusation: if something looks suspicious, treat it as a question of intent, not a guaranteed issue.

---

### Code Interpretation and Avoiding False Positives

Code is not always self-contained; external variables, configuration, or runtime context may be required and intentional. Do not assume a file, function, or class is incorrect simply because it appears unused or indirect — it may be invoked dynamically or via dependency injection. Avoid flagging patterns that are idiomatic for the language, shell, or environment. Do not mistake deliberate flexibility, extensibility, or debugging scaffolding for dead code.

Do not raise issues based on personal preference or stylistic bias; rely only on documented standards. Do not assume that commented-out code, alternate variable names, or debugging helpers are mistakes — they may be intentionally retained. Do not treat dynamic behaviour, late binding, or runtime composition as errors unless they violate explicit best-practice rules. When in doubt, trace usage, check patterns, and compare against the project's architectural principles before reporting an issue.

---

### Path Handling and Structure Assumptions

Do not flag path logic as incorrect solely because it looks unusual; verify whether the project structure or environment requires it. Only raise concerns when path logic is fragile, unreadable, or violates explicit best-practice rules. Prefer explicit, readable configuration or clearly defined root variables over implicit filesystem inference.

---

### Communication and Reasoning Discipline

Do not repeat the user's reasoning or present it as if the agent discovered it — treat user deductions as established facts and continue from them. Avoid filler, meta-commentary, or unnecessary explanations; responses must be precise, minimal, and focused on the user's intent. Do not assume emotional state, intent, or context beyond what is explicitly stated. Avoid robotic repetition or formulaic phrasing. Think carefully before responding — unusual patterns or phrasing may be intentional and must not be corrected without evidence. Always prioritise clarity, correctness, and intent over verbosity.

---

### Pattern Awareness

Match observations to the closest documented principle across general, AI, planner, coder, and reviewer best-practice files. Recognise patterns such as functional programming, OOP principles, interface isolation, domain-based interfaces, dependency injection, repository patterns, and architectural boundaries. Treat best-practice documents as a conceptual index: map behaviour, structure, or concerns to the appropriate category before forming conclusions.

---

## Documentation and Project Structure

### Agent File Responsibilities

All agents read `.github/best-practice/*.md` and `.github/copilot-instructions.md`. The **Planner** additionally reads `.github/planner-instructions.md` and context from `reference/documentation/planning/issues.md` and `reference/documentation/change-log.md`; it writes only to `reference/documentation/planning/plan.md` and `out/planner.notes`. The **Coder** reads `.github/coder-instructions.md` and `reference/documentation/planning/plan.md`; it writes code, tests, and `reference/documentation/change-log.md`, with notes in `out/coder.notes`. The **Reviewer** reads `.github/reviewer-instructions.md` and all source, test, config, script, and documentation files; it writes only `reference/documentation/planning/issues.md`, with notes in `out/reviewer.notes`. No agent may modify `.github` files.

---

### Permanent vs Temporary File Storage

All permanent documentation must be stored under `reference/`. All temporary, generated, or ephemeral files must be stored under `out/`. Notes files live only in `out/` and must never be stored under `reference/`, read by other agents, committed, or used as documentation.

The standard permanent structure is:

```
reference/
├── documentation/
│   ├── change-log.md
│   ├── machine-capabilities.md
│   ├── project-overview.md
│   ├── archive/
│   └── planning/
│       ├── issues.md
│       └── plan.md
├── scripts/
└── resources/

out/
├── planner.notes
├── coder.notes
├── reviewer.notes
└── scratch/
```

---

### Changelog Best Practices

Every changelog entry must be a single Markdown list item with no line breaks, no wrapping, and no indentation. Use the canonical format:

```
- YYYY-MM-DDTHH:MM:SSZ - [Category/Task] - file1.py, file2.py - Summary of change - +X/-Y
```

Use consistent ` - ` separators. No Markdown formatting inside entries. No commentary, no future plans, no explanations. Newest entry always at the top. Strictly append-only; never rewrite history. Aim for fewer than 240 characters per entry. Each entry must contain exactly five ` - ` separators, a valid ISO-8601 UTC timestamp, and a `+X/-Y` delta.

---

### Documentation Standards

Notes, logs, and documentation must place newest information at the top. Documentation must be human-readable paragraphs describing what was done, not future plans. When generating documentation intended for other AI systems, use single continuous paragraphs per list item, avoid heavy Markdown formatting, keep headings minimal, and ensure maximum embedding consistency and retrieval quality. Permanent documentation lives under `reference/documentation/**/*`. Resources live under `reference/resources/**/*`. PowerShell scripts live under `reference/scripts/**/*`.
