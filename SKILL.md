---
name: audit
description: Run a full 10-phase codebase audit — static analysis, dependency scan, test coverage & verification, architecture & decomposition, defensive programming, runtime risks, naming & clarity, code review readiness — then produce a prioritized report and fix all issues directly (CRITICAL first, then WARNING, then SUGGESTION). Informed by Code Complete 2 principles adapted for agentic coding. Use when the user says "run audit", "audit the codebase", or "check code quality".
argument-hint: "[--skip-tools] [--phase N]"
---

# Codebase Audit — CC2 Principles for Clean AI Coding

Execute an audit of the current project. Phases 1–8 are analysis only. Phase 9 produces the report. Phase 10 fixes all issues directly — CRITICAL first, then WARNING, then SUGGESTION — committing after each pass.

**Optional arguments:**
- `--skip-tools` — Skip external tool phases (1–3) and run only code analysis phases (4–8).
- `--phase N` — Run only phase N and report findings for that phase.

## Prerequisites

1. Read `CLAUDE.md` and any project documentation files that exist before starting.

---

## Phase 1 — Static Analysis

1. **Linters:** Check if `ruff` is installed (`pip install ruff` if missing). Run `ruff check --select F,W --no-fix .` to detect unused imports, unused variables, dead code, and unreachable branches.
2. **Type checker:** Check if `pyright` is installed (`pip install pyright` if missing). Run `pyright .` for type errors.
3. **Hardcoded secrets:** Search all `.py` files for patterns like `token\s*=\s*["'][^"']+`, `secret`, `password`, `api_key` assigned to string literals. Ignore test files with obviously fake values. Verify `.env` is in `.gitignore`.

**Record:** Every finding with file, line, rule ID, and description. Tag: `STATIC`.

---

## Phase 2 — Dependency Audit

1. **CVE scan:** Check if `pip-audit` is installed (`pip install pip-audit` if missing). Run `pip-audit -r requirements.txt` (or equivalent for the project's dependency file).
2. **Import cross-reference:** Parse all `import` and `from ... import` statements across every `.py` file. Exclude stdlib and internal package imports. Flag any third-party package imported but not in the dependency file, and vice versa.
3. **Outdated packages:** Check for available upgrades via `pip install --dry-run --upgrade <package>`.

**Record:** Each CVE with package, version, CVE ID, and fix version. Each import mismatch. Tag: `DEPS`.

---

## Phase 3 — Test Coverage & Verification

*CC2 principle: Agent Output Verification — verify correctness mechanically, not by reading every line.*

### Existing coverage checks

1. Check if `pytest` and `pytest-cov` are installed. Install if missing.
2. Run `pytest -v --tb=short`. Report pass/fail/skip/error counts.
3. If tests exist, run `pytest --cov=<source_package> --cov-report=term-missing` to identify uncovered lines. Adapt `<source_package>` to the project's main package.
4. For every source module, classify test coverage status:
   - **Covered** — has a corresponding test file with meaningful tests.
   - **Uncovered (Easy)** — pure logic, no mocks needed.
   - **Uncovered (Medium)** — needs mocks or test fixtures.
   - **Uncovered (Hard)** — external APIs or UI code, validated manually.
5. Flag any tests that assert on implementation details rather than behavior.

### Verification quality checks

6. **Property-based tests:** Search for `hypothesis` imports or `@pytest.mark.parametrize` with >3 test cases. Flag modules with complex logic (branching, math, parsing) but only example-based tests as SUGGESTION.
7. **Assertion density:** In each test function, count `assert` statements (including `pytest.raises`, `assertEqual`, etc.). Flag tests with 0 or 1 assert as WARNING — weak verification.
8. **Schema validation at boundaries:** Look for functions that accept `dict`, `Any`, or `**kwargs` as primary input. Check whether they validate with pydantic `BaseModel`, `TypedDict`, `dataclass`, or explicit key checks before use. Flag unvalidated dict inputs as WARNING.
9. **Output contracts:** Find functions that return `dict`, `list`, or `tuple` without specific type parameters in their annotation. Flag as SUGGESTION — these are unusable as agent specifications.

**Record:** Coverage percentage. List of uncovered files by difficulty tier. Verification findings. Tag: `TEST`.

---

## Phase 4 — Architecture & Decomposition

*CC2 principles: Problem Decomposition (Ch. 5), Coupling & Cohesion (Ch. 6) — clean module boundaries enable agent subtask routing and limit blast radius.*

### Existing architecture checks

1. **Circular imports:** Build the import graph for all project modules. Detect cycles. Flag deferred imports inside functions (potential circular-avoidance smell).
2. **Long functions:** Flag any function or async function with **more than 50 lines**. Include file, line, function name, and line count.
3. **Long files:** Count lines in every `.py` file. Flag any **over 300 lines** as a splitting candidate. Note files **over 200 lines** as approaching the threshold.
4. **Repeated logic:** Search for patterns that appear in 3+ files — same function call sequences, same string formatting, same list comprehension shapes. Note all locations and suggest whether abstraction is warranted.

### Decomposition & coupling checks

5. **Fan-in/fan-out analysis:** For each module, count how many other internal modules import it (fan-in) and how many internal modules it imports (fan-out).
   - Flag any module with **fan-out > 5** as WARNING — overly coupled, knows too much.
   - Flag modules with **fan-in > 8 AND fan-out > 3** as WARNING — hub/god module.
   - Report the average fan-out across all modules as a key metric.
6. **God object detection:** For each class, count public methods (excluding dunder), instance attributes, and total lines.
   - Flag classes with **>10 public methods** OR **>15 instance attributes** OR **>200 lines** as WARNING.
   - These resist agent subtask routing because responsibilities are tangled.
7. **Module boundary clarity:** For each `__init__.py`, check whether it defines `__all__`.
   - Flag packages where `__init__.py` has no `__all__` and re-exports symbols as SUGGESTION.
   - Without explicit `__all__`, agents cannot distinguish public API from internals.
8. **Layer violation check:** Detect common layer directories (`models/`, `views/`, `api/`, `routes/`, `services/`, `handlers/`, `db/`, `core/`, `utils/`). If the project uses recognizable layers, check for imports that skip layers (e.g., a view importing directly from db, a model importing from api). Flag each violation as WARNING. Skip this step if no layer structure is detected.
9. **Parameter count:** Flag functions with **>5 parameters** (excluding `self`, `cls`, `*args`, `**kwargs`) as SUGGESTION. High parameter counts indicate the function is doing too much or its inputs should be grouped.

**Record:** Each finding with file, line, metric, and recommendation. Tag: `ARCH`.

---

## Phase 5 — Defensive Programming & Error Handling

*CC2 principle: Defensive Programming (Ch. 8) — in agentic pipelines, tools receive malformed output from upstream model steps, not just bad human input.*

### Existing error handling checks

1. **Bare excepts:** Search for `except:` with no exception type specified.
2. **Swallowed errors:** Find `except` blocks whose body is only `pass` or sets a variable to `None` with no logging.
3. **Broad exception catches:** Find `except Exception` blocks. Classify each as:
   - **Acceptable** — UI error boundary that shows the error to the user.
   - **Suspect** — catches Exception but does not log or surface it.
4. **Async with no error boundary:** Find every `async def` function. Flag any without a `try/except` — an unhandled exception in an async handler can crash the event loop or leave the UI broken.

### Defensive programming checks

5. **Guard clause coverage:** For each public function (no leading underscore), check whether parameters typed as `Any`, `dict`, `Optional[X]`, or untyped are validated before use. Flag functions that use such parameters in operations (indexing, attribute access, arithmetic) without prior type/None/range checks as WARNING.
6. **Assertion density in production code:** Search for `assert` statements in non-test `.py` files. Flag modules with **>100 lines of logic and zero assertions** as SUGGESTION — missing runtime contracts. In agentic pipelines, assertions in production code are *desirable* as runtime contracts.
7. **Upstream output trust:** Find patterns where the return value of one function is passed directly to another without intermediate validation — specifically `result = foo(); bar(result)` where `foo` returns `Optional` or `Union` types but `result` is not checked. Flag as SUGGESTION.
8. **Fail-fast ratio:** Count how many `except` blocks re-raise (`raise` or `raise ... from`) vs. how many return a default value or `None`. Report the ratio. Flag if **>60% fail-silent** as WARNING — errors are being hidden, making agentic debugging impossible.

**Record:** Each finding with file, line, handler type, and severity. Tag: `DEFENSE`.

---

## Phase 6 — Runtime Risk Scan

### Existing runtime checks

1. **Timeouts:** Find every HTTP call (`requests.get/post`, `httpx`, `aiohttp`, `urllib`). Verify each has a `timeout=` parameter. Flag any that don't.
2. **Retry logic:** Check if any HTTP client has retry/backoff logic. Flag if no retry mechanism exists for external API calls.
3. **Unbounded loops:** Find all `while` loops. Check if each has a clear termination condition. Flag any that depend solely on external state.
4. **File I/O without error handling:** Find `open()`, `read_text()`, `write_text()` calls. Check if they're inside a try/except. Flag unguarded ones.
5. **Destructive operations:** Find SQL `DELETE`, `DROP`, `TRUNCATE`, `os.remove`, `shutil.rmtree`. Flag `ON CONFLICT DO UPDATE` if there's no audit logging.
6. **DB writes without error handling:** Find `cur.execute` or `conn.commit` without a surrounding try/except.

### Additional runtime checks

7. **Concurrency safety:** Search for global mutable state (`global` keyword, module-level mutable variables like `_cache = {}`, `_registry = []`). Flag any that are accessed in code that also uses `async` or `threading` without a lock as WARNING.
8. **Resource cleanup:** Find `open()`, database connections, HTTP sessions, or file handles not used as context managers (`with` statements). Flag as SUGGESTION.

**Record:** Each finding with file, line, risk type, and recommendation. Tag: `RUNTIME`.

---

## Phase 7 — Naming & Clarity

*CC2 principles: Naming & Clarity (Ch. 11), Prompt as Specification — function names, argument names, and docstrings are the interface agents use to reason about whether and how to call a function.*

1. **Single-letter and abbreviated names:** Search for function parameters, local variables, and function names that are single characters (excluding conventional `i`, `j`, `k`, `x`, `y`, `n`, `_`, `e` in except) or common abbreviations (`mgr`, `ctx`, `cfg`, `tmp`, `ret`, `val`, `buf`, `idx`, `fn`, `cb`). Severity: SUGGESTION for locals, WARNING for function parameters and function names.
2. **Boolean parameter ambiguity:** Find functions with `bool` parameters. Flag any where the parameter name does not start with a predicate prefix (`is_`, `has_`, `should_`, `can_`, `allow_`, `enable_`, `use_`, `include_`, `force_`, `skip_`, `with_`). A call like `process(data, True)` is opaque to agents. Severity: SUGGESTION.
3. **Docstring coverage:** For each public function and class, check whether a docstring exists. Report the percentage as a key metric. Flag any public function with **>3 parameters and no docstring** as WARNING.
4. **Type hint completeness:** For each function, check whether all parameters and the return type have annotations. Report the percentage as a key metric. Flag any public function missing annotations as SUGGESTION.
5. **Ambiguous return types:** Flag functions whose return annotation is `Any`, bare `dict`, bare `list`, `tuple` (without element types), or `Optional[Any]` as SUGGESTION. These are unusable as agent specifications.
6. **Naming consistency:** Within each module, check for mixed verb prefixes for the same pattern — e.g., some functions use `get_X` while others use `fetch_X` or `retrieve_X` for similar operations. Flag inconsistencies as SUGGESTION.

**Record:** Each finding with file, line, name, and recommendation. Tag: `NAMING`.

---

## Phase 8 — Code Review Readiness

*CC2 principle: Code Review & Inspections (Ch. 21) — when agents generate code at scale, human review becomes a sampling and oversight problem. Code must be structured for efficient review.*

1. **Multi-statement lines:** Search for lines containing `;` that separate multiple statements (exclude `for` loop headers, string literals, and comments). Flag each as SUGGESTION.
2. **Magic numbers and strings:** Search for numeric literals (excluding `0`, `1`, `-1`, `2`, `100`, `1000`) and string literals used in conditionals or arithmetic that are not defined as named constants. Flag as SUGGESTION.
3. **TODO/FIXME/HACK/XXX inventory:** Grep for these markers. For each, extract the surrounding context. Report as a table with file, line, marker, and content. Severity: informational (no severity assigned, included for reviewer awareness).
4. **Comment density:** For each file, calculate the ratio of comment lines (lines starting with `#` after stripping whitespace) to total non-blank lines.
   - Flag files with **<5% comment ratio AND >100 lines** as SUGGESTION (insufficient context for reviewers).
   - Flag files with **>40% comment ratio** as SUGGESTION (possible over-commenting or commented-out code).
5. **Commented-out code detection:** Search for `#`-prefixed blocks (3+ consecutive commented lines) that look like code — containing `=`, `def `, `class `, `import `, `return `, `if `, `for `, `while `. Flag each block as SUGGESTION.
6. **Function ordering:** In each module, check whether public functions (no underscore prefix) are grouped together vs. interspersed with private helpers. Flag files where public and private functions alternate frequently (>3 alternations) as SUGGESTION — makes the public surface hard to scan during review.

**Record:** Each finding with file, line, and description. Tag: `REVIEW`.

---

## Phase 9 — Report

Produce a single prioritized report with the following structure:

### Severity criteria

- **CRITICAL** — Will fail in production, security vulnerability, or data loss risk. Must fix before shipping.
- **WARNING** — Tech debt that is likely to cause bugs, hinder debugging, or block future work. Should fix soon.
- **SUGGESTION** — Cleanup, readability, or best-practice improvement. Fix when convenient.

### Category tags

Each finding is tagged with the phase it came from:

| Tag | Phase | Covers |
|-----|-------|--------|
| `STATIC` | 1 | Lint errors, type errors, hardcoded secrets |
| `DEPS` | 2 | CVEs, import mismatches, outdated packages |
| `TEST` | 3 | Coverage gaps, weak verification, missing contracts |
| `ARCH` | 4 | Decomposition, coupling, god objects, layer violations |
| `DEFENSE` | 5 | Error handling, guard clauses, fail-fast ratio |
| `RUNTIME` | 6 | Timeouts, unbounded loops, I/O safety, concurrency |
| `NAMING` | 7 | Clarity, docstrings, type hints, ambiguous types |
| `REVIEW` | 8 | Magic numbers, commented-out code, function ordering |

### Report format

```
## Audit Report

**Project:** <name>
**Date:** <date>
**Scope:** <N files, N lines of Python>

### Summary

| Category | Critical | Warning | Suggestion |
|----------|----------|---------|------------|
| STATIC   | 0        | 2       | 1          |
| DEPS     | 1        | 0       | 0          |
| TEST     | 0        | 3       | 2          |
| ARCH     | 0        | 1       | 4          |
| DEFENSE  | 0        | 2       | 1          |
| RUNTIME  | 1        | 0       | 0          |
| NAMING   | 0        | 1       | 5          |
| REVIEW   | 0        | 0       | 3          |
| **Total**| **2**    | **9**   | **16**     |

### Key Metrics

- Test coverage: XX%
- Docstring coverage: XX%
- Type hint coverage: XX%
- Avg fan-out per module: X.X
- Modules with fan-out > 5: N
- Classes with > 10 public methods: N
- Functions with > 5 params: N
- Fail-silent error ratio: XX%
- Assertion density (prod): X per 100 lines

### Findings

| # | Severity | Category | File | Line | Description | Recommended Fix |
|---|----------|----------|------|------|-------------|-----------------|
| 1 | CRITICAL | DEPS     | -    | -    | CVE-2024-... in pkg v1.2 | Upgrade to v1.3 |
```

Group by severity (CRITICAL first, then WARNING, then SUGGESTION). Within each severity, order by category tag.

### TODO/FIXME Inventory (informational)

If Phase 8 step 3 found any markers, include them as a separate table at the end of the report — not as findings with severity, just as a reference for the reviewer.

---

## Phase 10 — Fix Everything

After producing the report, fix all findings directly. Work through them in three passes by severity. **Between each pass, tell the user which severity level you just finished and which you're starting next** so they can follow along.

### Pass 1 — CRITICAL findings
Tell the user: "Fixing CRITICAL issues first."

Fix every CRITICAL finding. These are security vulnerabilities, production failures, and data loss risks — they all get fixed now. This includes dependency upgrades, secret removal, missing error boundaries on destructive operations, and anything else tagged CRITICAL.

When done, tell the user how many CRITICAL findings were fixed and commit the changes.

### Pass 2 — WARNING findings
Tell the user: "Moving on to WARNING-level issues."

Fix every WARNING finding. These are tech debt items that cause bugs or block future work — long functions that need splitting, god objects that need decomposing, missing timeouts, swallowed errors, coupling issues, etc. For larger refactors (splitting files, restructuring modules), do the work — don't just note it.

When done, tell the user how many WARNING findings were fixed and commit the changes.

### Pass 3 — SUGGESTION findings
Tell the user: "Now cleaning up SUGGESTION-level items."

Fix every SUGGESTION finding. These are readability and best-practice improvements — adding type hints, docstrings, renaming ambiguous variables, removing commented-out code, extracting magic numbers to constants, etc.

When done, tell the user how many SUGGESTION findings were fixed and commit the changes.

### After all passes
Report a final summary: total findings fixed per severity, total commits made, and any findings that genuinely could not be fixed (with an explanation of why — not just "it's hard").

---

## Notes

- Phases 1–8 are analysis only — no code changes until Phase 10.
- All tool installations use the current Python environment.
- If a tool fails to install or run, note it in the report and continue with remaining phases.
- The audit assumes the working directory is the project root.
- Phases 7 and 8 require judgment — err toward SUGGESTION severity to avoid noise.
- The CC2 principles informing each phase are noted for context, not as rules to cite in findings. Findings should be concrete and actionable.
