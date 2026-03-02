+++
title = "MPC Project Analyser"
description = "A deterministic, batch-based Python analysis engine that generates compact structural metadata to enable AI agents to plan software changes safely and cost-effectively."
tags = ["AI", "Python", "Architecture"]
aliases = ["/projects/optomising-ai-credits/", "/projects/optimising-ai-credits/"]
+++

## MPC Project Analyser

The **MPC Project Analyser** is a deterministic, batch-based analysis engine written in Python. It bridges the gap
between expensive, high-reasoning AI models and cost-effective execution models. Rather than feeding raw source code
to every AI agent, the analyser generates compact structural metadata — AST signatures, file trees, documentation
maps — that enables agents to plan software changes safely and efficiently without consuming unnecessary context.

The driving insight is that most AI credits are wasted when a single premium model does everything: deep reasoning,
mechanical coding, test generation, and repetitive fixes. By assigning distinct **personas** to different model tiers
and switching between them deliberately, the same quality of output is achievable at a fraction of the cost. Premium
models think — they review completed work and produce architectural plans. Free or standard models build — they
execute the plan, write code, and update logs. Each phase runs in its own **new chat session** so context stays
clean and no premium credits are spent on deterministic work a free model can handle.

---

## Engineering Principles

The analyser is built on principles that make its output reliable enough for AI consumption.

**Structure-first** means the system analyses the *shape* of code — classes, functions, namespaces, call signatures
— rather than its raw text. This produces compact, token-efficient output that communicates intent without noise.
**Deterministic output** ensures that given the same input the analyser produces bit-exact output every time, which
is a prerequisite for AI agents that need to compare runs or detect drift. **Interface-driven design** means all
dependencies are injected via `Protocol` abstractions; the core never references concrete classes directly.
**Single Responsibility** is enforced at the package level: `capabilities`, `configuration`, and `output` are
separate domains. **Zero side effects** means the analyser reads code but never modifies it; data goes to `stdout`
and logs go to `stderr`.

---

## Architecture

The system is partitioned into five domains that build on each other cleanly.

The **error and logging infrastructure** was established first. A typed exception hierarchy —
`ContentAnalyserError`, `ConfigurationError`, `FileSystemError` — ensures failures are granular and catchable.
Logging is centralised and idempotent, writing exclusively to `stderr` so `stdout` remains clean for JSON output.

The **configuration system** treats configuration as a domain model rather than a dictionary. `AnalysisConfig` and
`OperationConfig` provide type-safe wrappers around user input. A validated JSON loader acts as the gateway,
applying the fail-fast principle by rejecting invalid schemas at startup before any analysis begins.

The **capability engine** is the core of the system and is built on the Strategy pattern. A registry of named
capabilities allows the CLI to be open for extension without modification. `FileTree` generates nested directory
maps to visualise project layout. `FileWalker` extracts metadata — line counts, file sizes, types — for resource
estimation. `CodeTree` uses Python's `ast` module to parse source into high-level signatures (classes, methods,
functions, parameters) while skipping implementation bodies entirely, minimising token usage for downstream AI
consumers.

The **CLI and output layer** is intentionally thin. A `main.py` orchestrator using `argparse` supports two modes:
schema mode outputs the validator schema when called with no arguments; analysis mode executes the requested
operations when passed a `--config` flag. A `JsonResultFormatter` ensures all data is serialised to strictly
formatted, deterministic JSON.

---

## AST Metadata

The Abstract Syntax Tree (AST) is a structured representation of source code produced by the language parser before
compilation or execution. Rather than treating code as raw text, the AST exposes it as a tree of named nodes —
modules, classes, functions, parameters, decorators, return types — each with precise location information. The
analyser walks this tree and extracts only the structural signatures, discarding all implementation bodies. The
result is a compact, token-efficient summary of what the code *does* without encoding how it does it.

This matters for AI consumption because large language models have finite context windows. Sending raw source files
fills that window with implementation detail that is irrelevant to planning. An AST summary for a module with ten
classes and fifty methods might consume a few hundred tokens where the original source would consume tens of
thousands. The planner can reason about the entire codebase structure in a single context, identify what needs to
change, and write precise subtasks — all without ever seeing a line of implementation.

The metadata produced by `CodeTree` for each source file includes the module path, top-level imports, and for each
class: its name, base classes, decorators, and a list of methods with their signatures, parameter names, type hints,
and return types. Functions defined at module level are captured with the same detail. This is sufficient for a
premium model to assess architecture, identify violations, and plan interfaces-first changes without ambiguity.

---

## The Persona Workflow

The analyser is designed to be used alongside a three-persona AI workflow. Each persona runs in a new chat session
with the appropriate model tier, and the instruction files below are loaded at the start of every session to ensure
consistent engineering standards regardless of which model is running.

- **[Reviewer](/projects/ai/reviewer-instructions/)** — premium model; reads all source, tests, and docs; writes a prioritised `issues.md` report; strictly read-only.
- **[Planner](/projects/ai/planner-instructions/)** — premium model; reads issues and change-log; writes a deterministic `plan.md`; never touches code.
- **[Coder](/projects/ai/coder-instructions/)** — free or standard model; executes the plan exactly; updates the change-log after every subtask.

The supporting reference files that every persona reads:

- **[copilot-instructions](/projects/ai/copilot-instructions/)** — universal coding standards applied across all sessions.
- **[best-practice](/projects/ai/best-practice/)** — the engineering handbook covering patterns, testing, architecture, and package structure.

### Reviewer Prompt

```text
You are the Reviewer. Read the reviewer-instructions, copilot-instructions, and .github/best-practice.md first.
Then read all source, test, config, and documentation files including plan.md and change-log.md. Produce a
prioritised best-practice report in reference/documentation/planning/issues.md covering architectural violations,
design patterns, package structure, and quality issues — with severity, violated standard, impact, and next steps
for each. Your role is strictly read-only. Do not modify code, tests, plans, or create any file other than issues.md.
```

### Planner Prompt

```text
You are the Planner. Read the planner-instructions, copilot-instructions, and .github/best-practice.md first.
Then read reference/documentation/planning/issues.md, reference/documentation/change-log.md, and
reference/session-summary.md. Remove completed tasks from the existing plan and produce
reference/documentation/planning/plan.md with 3–10 milestone tasks and no more than 100 subtasks. Write every
subtask interfaces-first using the strict subtask format. Resolve all issues from issues.md before adding new
work. Do not modify any file except plan.md. Finish the plan with: "Ready for Coder execution".
```

### Coder Prompt

```text
You are the Coder. Read the coder-instructions, copilot-instructions, and .github/best-practice.md first. Fix
all issues in reference/documentation/planning/issues.md, then execute every task in
reference/documentation/planning/plan.md exactly as written — interfaces before implementations, one task at a
time. Update reference/documentation/change-log.md with an ISO-8601 entry after each subtask. Do not modify
planning files, add unrequested features, or ask questions.

Before moving to the next task, verify your work against the coder-instructions: correct interfaces, no
placeholder comments, tests written and passing, change-log updated, and size limits respected. Summarise
completed subtasks and confirm before starting the next task.
```

---

## Developer Guidelines

The project layout mirrors the domain structure. `src/content_analyser/` contains source code partitioned by
domain. `tests/` mirrors the source structure exactly — one test class per source class, in the same relative
path. `reference/` holds documentation and planning artefacts.

Tests follow AAA (Arrange, Act, Assert) with one test method per use case. Internal logic is never mocked; only
external systems and expensive operations are isolated. Temporary test artefacts go to `out/tests/<domain>/` to
keep the workspace clean.

The contribution workflow is: check `issues.md` for prioritised tasks, follow `plan.md` strictly, update
`change-log.md` with every significant change.

---

## Roadmap

The functional core is complete. Current focus is test hygiene, docstring coverage on public interfaces, and
decoupling the CLI from concrete implementations. Planned extensions include multi-language `CodeTree` support
(TypeScript, Java, C#), a Markdown heading analyser for documentation structure extraction, further JSON
compression for token optimisation, and a `pip`-publishable distribution workflow.

---

## Skills

**Languages & Tools**
Python | AST Parsing | argparse | JSON Schema Validation | Pytest

**Architecture & Design**
Strategy Pattern | Capability Registry | Interface-Driven Design | Domain-Driven Packages | Fail-Fast Configuration

**AI Workflow**
Persona-Driven Development | Premium vs Free Model Routing | Context Window Optimisation | Token-Efficient Metadata

**Code Quality**
Pure Functions | Zero Side Effects | Typed Exception Hierarchy | Structured Logging | Deterministic Output

**Testing**
AAA Pattern | Fast-Failing Tests | Mirrored Test Structure | One Test per Use Case | Domain-Grouped Test Classes

**Process**
Interfaces-First Planning | Change-Log Discipline | Issues-Driven Iteration | Three-Persona Review Cycle

