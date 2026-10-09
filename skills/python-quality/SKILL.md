---
name: python-quality
description: Python quality standards and gates. Use when changing or integrating Python code, tests, dependencies, or tooling, or when conducting a code review.
---

# Python Quality Profile

Apply this profile inside the active **host workflow**: the engineering skill that owns process and sequencing. This profile owns Python policy only. The host retains planning, TDD sequencing, git operations, subagents, review orchestration, reporting, and its own overall completion criteria.

## Terms

- **Gate**: the required verification checkpoint `uv run poe check`.
- **Verified state**: repository files after a successful gate run, provided no repository files have changed since that run. Any later repository-file change invalidates it.
- **Verified handoff**: handing a verified state back to the host workflow.

## Authority

Resolve decisions in this order:

1. The host workflow owns process and sequencing.
2. Repository tool configuration owns mechanical behavior.
3. Repository standards and ADRs own project judgment and architecture.
4. The selected references below supply Python judgment where the repository is silent.
5. A contradiction in the above is **repository drift**; report it rather than choosing silently.

## Route references

Before implementation or review, assess **every row** and read **every match**. Reference paths are relative to this skill's directory; resolve them to absolute paths before passing them to another agent.

| Affected material or decision | Read |
| --- | --- |
| Python source, including style, annotations, strings, or imports | `references/python-style.md` |
| Tests are affected, or the active `tdd` cycle requires a Python test | `references/testing.md` |
| Generics, protocols, overloads, decorators, or advanced annotations | `references/advanced-typing.md` |
| Public/private names, exports, warnings, deprecations, or compatibility | `references/visibility.md` |
| Iteration, comprehensions, caching, pattern matching, or context managers | `references/idioms.md` |
| Comments, docstrings, TODOs, or explanatory source prose | `references/comments.md` |
| Equality, hashing, representations, containers, or data-model methods | `references/magic-methods.md` |
| Creating, moving, splitting, or organizing modules | `references/file-organization.md` |
| Package layout, imports, `__init__.py`, or package exports | `references/project-structure.md` |
| Assertions, breakpoints, tracing, or temporary diagnostics | `references/debugging.md` |
| Exceptions, validation failures, or exception chaining | `references/error-handling.md` |
| Dataclasses, typed dictionaries, named tuples, or value objects | `references/data-structures.md` |
| Classes, constructors, factories, properties, inheritance, or composition | `references/class-design.md` |
| Pydantic v2 models, validators, settings, or serialization | `references/pydantic.md` |
| Environment variables, configuration files, or application settings | `references/configuration.md` |
| Filesystem paths or file operations | `references/pathlib.md` |
| Threads, processes, locks, queues, or shared state | `references/concurrency.md` |
| Coroutines, tasks, event loops, or async context managers | `references/async.md` |
| Logging in application or library code | `references/logging.md` |
| Untrusted input, sensitive data, subprocesses, deserialization, cryptographic randomness, or dependency hygiene | `references/security.md` |
| NumPy arrays, dtypes, shapes, vectorization, or numerical APIs, when NumPy is present or being added as a runtime dependency or optional extra | `references/numpy-types.md` |
| Matplotlib figures, axes, or plotting APIs, when Matplotlib is present or being added as a runtime dependency or optional extra | `references/plotting.md` |

The security floor in `references/security.md` is not overridable. A conflict that would expose secrets or sensitive data, execute or deserialize untrusted input unsafely, or create an equivalent concrete vulnerability blocks the work.

## Implementation mode

Apply selected references while changing Python-relevant material. At the single implementation handoff:

1. Inspect repository status and record repository-file status.
2. Run `uv run poe check` from the repository root.
3. Inspect status again. Gate-produced edits are implementation changes: retain, inspect, and include them in the diff.
4. Fix agent-owned failures—code, tests, formatting, typing, and local configuration—and rerun the gate as needed.
5. If dependencies are missing, derive the repository-appropriate `uv sync` command from its metadata, give that exact command to the user, and wait for confirmation. Treat infrastructure or external-service failures as blockers. If the `check` task is missing, direct the user to `/setup-python-quality`.
6. Report the exact gate command, pass/fail status, and whether it changed files.

Implementation mode completes with either a verified handoff or an explicitly reported blocker/unavailable gate; the host decides whether its workflow can complete.

## Integration mode

For mixed-language work, activate this profile when any Python source, tests, dependencies, or tooling changes. Apply references only to Python material, but run the complete repository-owned `uv run poe check`.

The host runs the gate on the integrated tree before code review, then after remediation or any other repository-file change. It may reuse the verified state after a read-only review when the tree stayed unchanged. Apply the same safety, failure, edit-inspection, and reporting contract as implementation mode.

Integration mode completes when the integrated tree has a verified handoff, or when its blocker is explicit for the host to resolve.

## Read-only code review mode

The code review host establishes a verified state before spawning reviewers when it can safely run the gate. A reviewer stays read-only and never runs the gate. If the repository is not in a verified state, continue the review and explicitly report the missing mechanical prerequisite.

Cite a selected reference for fallback findings. Skip tool-enforced findings only when the repository is in a verified state. Report repository drift and security-floor violations regardless of mechanical coverage.

Read-only code review mode completes when every applicable selected reference has been assessed and the report states whether the repository was in a verified state. Remediation is implementation mode and returns a verified handoff.
