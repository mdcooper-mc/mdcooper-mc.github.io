+++
title = "Best Practice"
description = "A practical engineering handbook covering design principles, patterns, testing discipline, and architecture guidelines."
+++

## The Foundation: Four Principles Every Codebase Needs

Good software starts with four principles that work together. The **Single Responsibility Principle (SRP)** says that every module, class, and function should have exactly one reason to change. When a unit handles multiple concerns, a change in one area risks breaking another, and tests become difficult to reason about because behavior spans unrelated paths. Keeping things focused makes refactoring safe and failures easy to isolate.

**Don't Repeat Yourself (DRY)** means every piece of domain knowledge has one unambiguous home in the codebase. Duplication is not just a code smell — it is a maintenance liability. When the same rule exists in three places, they will eventually diverge. Consolidate logic into shared abstractions and your system stays consistent as it grows.

**Keep It Simple** is the hardest principle to follow under pressure. Complexity compounds faster than most engineers expect. Every unnecessary abstraction, every premature optimization, and every clever trick adds cognitive load for the next person. Prefer the simplest solution that solves the actual problem, and resist the temptation to build for imagined futures.

**Interface-Driven Design** ties the others together. All code should depend on abstractions, not concrete implementations. Use interfaces as the type for variables, fields, parameters, and return values everywhere. Concrete types belong only at composition roots — the places where the application is wired together. This gives you loose coupling, easy substitution, clean mocking in tests, and enforced boundaries between domains. It is not a suggestion; it is the foundation of a maintainable system.

---

## Design Patterns

Patterns are vocabulary for solutions that recur across different problems. Knowing them means you can communicate intent clearly and apply proven structures instead of reinventing them. They fall into three classical groups.

**Creational patterns** control how objects come into existence. The **Factory Method** delegates the decision of which concrete type to create to a subclass or a strategy, keeping the calling code unaware of the implementation. The **Builder** constructs complex objects step by step, allowing different configurations without bloated constructors. The **Abstract Factory** creates families of related objects and is useful when the entire set of implementations must be switched together. **Prototype** clones existing objects when construction is expensive. **Singleton** ensures a single instance exists globally, though it should be used sparingly as it introduces hidden coupling.

**Structural patterns** describe how components are assembled. The **Adapter** wraps an incompatible interface so it fits where another is expected — invaluable when integrating third-party libraries. The **Decorator** adds behavior to an object at runtime without subclassing, composing capabilities cleanly. The **Facade** simplifies a complex subsystem behind a single, coherent interface. The **Composite** lets trees of objects be treated uniformly, which is the right model for hierarchical data. The **Proxy** controls access to another object, adding caching, authorization, or lazy initialization. **Bridge** separates abstraction from implementation so both can vary independently. **Flyweight** shares common state across many small objects to reduce memory pressure.

**Behavioral patterns** define how objects communicate. The **Strategy** pattern encapsulates interchangeable algorithms behind a common interface, making runtime behavior changes clean and testable. **Observer** notifies dependents automatically when state changes — the backbone of event-driven systems. **Command** wraps an action as an object, enabling queuing, undo, and logging. **Chain of Responsibility** passes a request along a chain of handlers until one processes it, decoupling senders from receivers. **State** lets an object's behavior change when its internal state changes without a mass of conditionals. **Template Method** defines an algorithm's skeleton in a base class and lets subclasses fill in the steps. **Visitor** separates an algorithm from the structure it operates on, letting you add operations without touching the objects. **Mediator** centralises communication between objects, reducing direct dependencies. **Memento** captures and restores state without exposing internals.

Beyond the Gang of Four, several architectural patterns are now standard. **Event Sourcing** stores state as an immutable sequence of events rather than current values, giving you a full audit trail and the ability to replay history. **CQRS** separates read and write models so each can be optimised independently. The **Repository** pattern abstracts data access behind a domain-facing interface, keeping storage details out of business logic. The **Service Layer** encapsulates business operations as explicit, reusable services. The **Saga** pattern coordinates distributed transactions across microservices using compensating actions rather than locks. **Null Object** eliminates null checks by providing a do-nothing implementation that satisfies the interface contract.

---

## Functional Programming

Functional programming is not an all-or-nothing paradigm — most of its value comes from applying its principles selectively inside any language or architecture.

The core idea is **immutability**: data does not change once created. New values are produced from old ones through transformation. This eliminates a wide class of bugs caused by shared mutable state, makes functions easy to reason about in isolation, and simplifies concurrency because there is nothing to lock.

**Pure functions** take inputs and return outputs with no side effects and no hidden dependencies on external state. Given the same inputs, they always produce the same output. Pure functions are trivially testable, safe to cache, and safe to run in parallel. They are the right tool for all data transformation logic.

**Higher-order functions** — functions that accept or return other functions — enable **composition**: building complex behavior by chaining small, focused transformations. Map, filter, and reduce are the workhorses. Pipelines built from these primitives are declarative, readable, and easy to extend by inserting a new step.

For error handling, **railway-oriented programming** models success and failure as two tracks flowing through a pipeline. An `Either` type (or equivalent) carries either a result or an error forward without exception-based control flow. This makes error paths explicit, composable, and impossible to accidentally ignore.

The **Functional Core / Imperative Shell** architecture applies these ideas at scale: keep all business logic in pure functions, push side effects — database writes, HTTP calls, file I/O — to the outermost shell. The core is fully testable without mocks. The shell is thin and straightforward.

---

## Object-Oriented Design

Object-oriented design is most effective when it mirrors the domain. **Entities** have identity that persists over time — a customer account is the same account regardless of how its attributes change. **Value Objects** are defined entirely by their attributes and carry no identity — a monetary amount, a date range, or a GPS coordinate. Treat value objects as immutable. **Aggregates** cluster related entities and value objects into a consistency boundary; the aggregate root is the only entry point for modifying anything inside it.

The most common misuse of inheritance is using it for code reuse rather than to express genuine is-a relationships. **Favour composition over inheritance**: assemble behaviour by combining small, focused interfaces rather than building deep class hierarchies. Hierarchies are rigid; composition is flexible.

OOP and functional programming are not opposites. Use pure functions for deterministic data transformations and use objects with interfaces for stateful or polymorphic behaviour. The two styles complement each other when applied to the problems they are suited to.

---

## Testing Discipline

Tests are not an afterthought — they are the primary mechanism for verifying that software does what it claims to do, and for maintaining that guarantee as the system changes.

Every test must follow **Arrange–Act–Assert**. Arrange sets up the context: inputs, mocks, and preconditions. Act executes exactly one behaviour. Assert verifies the outcome with specific, minimal expectations. This structure makes the intent of each test immediately readable and prevents hidden setup or assertion from obscuring failures.

Each test method should cover **one use case only**. No branching, no multiple scenarios, no "and also" assertions. A test is a short story: it has a single premise and a single conclusion. When a test fails, you should know immediately which behaviour broke without reading through a tangle of conditionals.

**Fast-failing tests** are a discipline, not just a performance concern. When the function under test contains logic that is irrelevant to the scenario being verified, use a mock that throws a controlled exception at the first opportunity. This avoids building deep mock chains for code paths that are not part of the use case, keeps tests small, and makes failures precise.

Tests should be **organised by domain**, not by technical layer. A test class represents a domain concept; its methods represent use cases within that concept. This mirrors the way real users think about behaviour and makes gaps in coverage obvious. The test directory should mirror the source directory exactly — one test file per source file, in the same relative path.

Aim for **70–90% coverage** on critical paths, edge cases, and error conditions. Do not write tests for trivial getters or well-covered infrastructure code — that effort is better spent on logic that actually makes decisions. Mock only external systems and expensive operations; avoid mocking internal logic, as that leads to tests that verify the mock rather than the code.

---

## Code Size and Complexity

Size limits are not arbitrary rules — they are indicators of structural problems. A file that grows beyond 200–500 lines usually contains mixed responsibilities. A class with more than ten methods is probably doing too much. A function longer than twenty lines almost certainly has a simpler decomposition waiting to be found.

**Cyclomatic complexity** is the number of independent execution paths through a function. High complexity means more test cases, more cognitive load, and higher bug probability. When complexity rises, it signals the need for decomposition: extract a strategy, introduce a state machine, or split into smaller functions.

Deep nesting is a form of complexity. More than two or three levels of indentation usually means the logic can be restructured using early returns, extracted helper functions, or a more appropriate pattern. Flat is better than nested.

---

## Package and Module Structure

Organise code by **domain**, not by technical role. A folder called `services/` or `controllers/` tells you nothing about what the code does; a folder called `payments/` or `identity/` tells you everything. Each domain package should contain everything relevant to that domain: its models, interfaces, implementations, factories, validators, and tests. This keeps related code together and makes the codebase navigable without knowing the architecture in advance.

Expose only what needs to be public. Everything else should be private or internal to its package. A minimal public surface area reduces coupling, makes refactoring safer, and makes the intended extension points explicit.

File names should not repeat the package name. The directory path already provides that context. A file named `payment_service.py` inside a `payments/` package repeats `payment` unnecessarily — `service.py` is sufficient and cleaner.

Avoid circular imports. They indicate that two modules are too tightly coupled and one of them is probably doing too much. Introduce a shared abstraction or invert the dependency.

---

## Architecture and Reliability

Good architecture makes the system easy to change without breaking things. The key properties are **low coupling** — components depend on abstractions, not on each other's internals — and **high cohesion** — everything in a component genuinely belongs together.

Design for **observability** from the start. Structured logging, metrics, and distributed tracing are not optional extras. When something goes wrong in production, they are the only tools available. Use contextual log entries that capture what the system was doing, not just that an error occurred.

**Graceful degradation** means the system continues to provide partial value when a dependency fails, rather than failing entirely. Design every integration point with a fallback: cached data, a default response, or a clear error to the caller. Fail fast on invalid input at the boundary; never let bad data propagate deeper into the system.

Externalise configuration. Hard-coded values are the fastest path to environment-specific bugs and the slowest path to fixing them. Use environment variables, configuration files, or a configuration service, and validate configuration at startup so problems surface immediately rather than at the worst possible moment.

---

## DevOps and Delivery

**Infrastructure as Code** means environments are defined, versioned, and reproduced the same way every time. Manual environment setup is a source of drift, inconsistency, and toil. If you cannot reproduce an environment from a repository checkout, you do not have a reliable environment.

**Quality gates** enforce standards before code reaches production. Linting, type checking, test coverage thresholds, and security scanning belong in the CI pipeline — not in code review, where they slow humans down with mechanical concerns that tools handle better.

**Blue-green and canary deployments** shift traffic gradually to new versions, allowing real-world validation before full rollout. Combined with **post-incident reviews** that focus on systemic causes rather than individual blame, they build a culture where failure is a learning event rather than a crisis.
