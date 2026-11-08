+++
title = "Copilot Instructions"
description = "Universal coding standards, naming, structure, and tooling rules applied by all AI personas."
+++

## General Behaviour

- Responses must be short, precise, actionable, and non-verbose.
- Prefer single-line, machine-parsable bullets for actionable items.
- Provide concrete examples and explicit expected outputs when needed for clarity.
- Select the single best option; research externally when correctness requires it.
- Never run scripts or commands without explicit permission; provide PowerShell commands only when allowed.
- Never create unnecessary files; only modify or create files explicitly required by the task.
- Never replace code with placeholder comments (e.g., `# ...existing code...`).
- Never expose internal tool names, rate limits, or runtime details.
- No emojis or images in any files or documentation.

## File Structure, Imports, and Code Organisation

- Keep imports at the top of files; do not use dynamic or in-function imports.
- Follow SOLID, DRY, KISS, and YAGNI.
- Use interfaces for all classes and services; call implementations only through interfaces.
- Use dependency injection for service interfaces; avoid unnecessary indirection layers.
- Prefer immutable parameters, fields, and variables where practical.
- Keep modules cohesive and loosely coupled with minimal public surface area.
- Structure code by domain and functionality; do not encode package names into file names.
- Update dependency files when adding dependencies.
- Do not add documentation inside code.

## Logging, Debugging, and Error Handling

- Use structured logging instead of `print`.
- Never hide exceptions; handle, log, or propagate them.
- Use events and well-defined models to decouple packages and pass data across boundaries.

## Tests and Quality

- Tests must follow AAA (Arrange, Act, Assert) with explicit `# Arrange`, `# Act`, `# Assert` comments.
- Tests must live alongside source code and mirror the package structure.
- Use one test class per source class and one test method per use case.
- Use mocks to isolate dependencies and fail fast.
- Test classes must extend `unittest.TestCase` and use its lifecycle and assertion methods.
- Aim for 70-90% coverage; test critical paths and edge cases, not trivial getters/setters.
- Apply marble testing for reactive logic where relevant.

## Documentation, Notes, and Logs

- Notes, logs, and documentation must place newest information at the top.
- Documentation must be human-readable paragraphs describing what was done (not future plans).
- Documentation for AI models must use single continuous paragraphs per list item with minimal formatting.
- Permanent documentation must live under `reference/documentation/**/*`.
- Resources must live under `reference/resources/**/*`.
- Temporary files must live under `out/**/*` and must not be committed.
- PowerShell scripts must live under `reference/scripts/**/*`.

## Issue Handling and Planning

- Do not bypass issues; fix them.
- Do not ignore failing tests; fix the code or request plan updates.
- If a file must be deleted, ask the user and provide the exact command.
- Never bypass the plan; request updates if the plan is incorrect.

## AI, RAG, and Model Usage

- Follow `.github/best-practice/AI.md` for all AI, RAG, and model-related behaviour.
- Assume interfaces are correct; treat them as the source of truth over implementations.
- Use lazy loading, batching, and device-aware execution.
- Use quantisation, mixed precision, and compilation when appropriate.
- Follow token optimisation and context budgeting rules.
- Use ONNX, TensorRT, or Torch Compile when beneficial.
- Never load a model per request; batch by model and process all items before unloading.
- Always respect machine capabilities defined in `machine-capabilities.md`.

## Documentation for Other AIs

- When generating documentation intended for other AI systems:
  - Use single continuous paragraphs per list item.
  - Avoid heavy Markdown formatting (tables, code blocks, nested bullets).
  - Keep headings minimal.
  - Ensure maximum embedding consistency and retrieval quality.

## Restricted Tools

- Do not attempt to use restricted tools: `run_in_terminal`.

## Final Rule

- These instructions apply to the AI only. They do not apply to the human operator.
