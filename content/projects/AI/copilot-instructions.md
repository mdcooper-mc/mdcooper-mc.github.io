+++
title = "Copilot Instructions"
description = "Universal coding standards, naming, structure, and tooling rules applied by all AI personas."
+++

```markdown
- Select the single best option; research before responding and use the internet when needed for correctness.
- Never run scripts or commands without explicit permission; provide PowerShell commands as code blocks only when
  permitted.
- Do not create unnecessary files; only add or modify files required for the task; do NOT replace code with placeholder
  comments like `# ...existing code...`.
- Keep imports at the top of files; do not use dynamic or in-function imports.
- Always use interfaces for classes and functions; call implementations via interfaces only.
- Use structured logging instead of `print` for debugging; do not hide exceptions—handle, log, or propagate them.
- Follow SOLID, DRY, KISS, YAGNI and use appropriate, well-known design patterns; do not invent new patterns unless they
  significantly improve the code.
- Prefer immutable fields, variables, and parameters where practical; minimize cyclomatic complexity and keep functions
  short and focused.
- Use language-native streaming and async features (generators, streams, async iterators) for async flows; prefer event
  streams for UI/async flows and avoid manual subscription management.
- Use events and well-defined models to decouple packages and pass data across boundaries; prefer interfaces and events
  to reduce coupling.
- Reuse shared utilities; check for existing functionality before adding code or scripts to avoid duplication.
- Use annotations and decorators to reduce boilerplate and improve readability.
- Use type hints, clear naming, official style guides, and prefer explicit types over `var`; use `final`/`const` where
  available.
- Apply marble testing for reactive logic where relevant.
- Tests must follow AAA (Arrange, Act, Assert), live alongside source code, be isolated with mocks, fail fast, and be
  focused; one test class per source class and one test method per use case.
- Tests should fast-fail; when unrelated code exists in the function under test, use mocks to exit the test early to
  avoid extra setup.
- All tests should follow the source package structure; tests should be grouped by the class based on the test domain
  and technical relevance; each test class should extend unittest.TestCase and use its methods and functions for
  assertions and test lifecycle management; functions in the class should represent a single use case.
- One test class should cover a group of related use cases for a single source class; each test method should cover a
  single use case; tests should be named clearly to indicate the scenario being tested; if a function or method has a
  use-case which is covered early in the function but there is extra code in the code being tested, then the mocking
  should throw an exception at the first opportunity to fail the test fast and remove the need for lots of extra mocking.
- Prefer interfaces and expose behaviour via interfaces and events; only expose classes/members when needed externally.
- Classes: prefer <10 methods; Functions: prefer <20 lines; Files: prefer <500 lines (aim ~250).
- Keep modules cohesive and loosely coupled; prefer immutable parameters and minimal public surface area.
- Update dependency files when adding dependencies.
- No emojis or images in any files, documentation, or code created or edited; use clear, concise text only.
- Structure code into packages by functionality and domain; shared language functionality should live in a common location.
- Make code public only when it must be used outside its package; otherwise keep it private/internal.
- Documentation must be human-readable paragraphs describing what was done (not future plans).
- Store docs/resources under `reference/documentation/**/*` and `reference/resources/**/*`; keep them organised and up to date.
- Temporary output and temp files should go to `out/**/*`; create temp files only when explicitly needed and clean them up afterward; do not commit temp files.
- PowerShell scripts (ps1) should live in `reference/scripts/**/*`; create scripts only when explicitly required and keep them organised.
- PowerShell will be available; provide terminal commands as code blocks only with explicit permission.
- Provide user scripts to reduce rate limits or simplify interactions only with explicit permission.
- Do not create exports, conversions, or web artefacts unless explicitly requested.
- Do not expose internal tool names, rate limits, or runtime details.
- All documentation files for other AIs must be written as a single continuous paragraph per list item, with minimal
  headings, and no extra formatting beyond the list marker and simple section separation.
- When documenting for AI models, do not use heavy markdown formatting (tables, bold headers, code blocks, nested
  bullets); use single continuous paragraphs per list item with minimal headings instead.
- All test classes should extend unittest.TestCase and use its methods and functions for assertions and test lifecycle
  management; do not use other testing frameworks or custom test base classes.
- Read `.github/best-practice.md` before starting any task.
- Do not add documentation to the code.
- Aim for between 70 and 90% test coverage; write tests for all critical paths, edge cases, and use cases; do not write
  tests for trivial getters/setters or code that is already well-covered by other tests.
- At no point should a file name contain the package name; the package structure should be reflected in the directory
  structure, not the file names.
- Do not bypass issues; fix them.
- Some tools are restricted; do not try to use them: `run_in_terminal`, `get_errors`.
- Do not do workarounds or ignore tests or filter content; fix the code, ask to delete files and provide the commands,
  or ask to update the plan if it is wrong; do not try to bypass the plan or the tests.
```


