---
name: python-tdd
description: Python-specific TDD implementation driver. Reads an issue, loads all listed Python references from the skill's own references/ directory, then drives a self-contained red-green-refactor loop using `uv run poe check` as the feedback step. Use when implementing an issue or any coding task in a Python project, or when the user says "implement this" or "work on issue" in a Python project.
---

# Python TDD

Implements a Python issue using test-driven development. Loads all listed Python references, then drives a red-green-refactor loop with `uv run poe check` as the feedback mechanism.

Prefer AFK execution: proceed without asking non-blocking questions, infer safe defaults from the issue/spec/codebase, and report decisions afterward. Stop only for genuinely blocking ambiguity or unsafe action.

Do not run commands that change git state: no `git add`, `git commit`, `git reset`, `git checkout`, `git switch`, `git clean`, or similar. Use git only for read-only inspection (`git status`, `git diff`, `git diff --staged`, `git rev-parse`, `git log`).

---

## Step 1 — Read the issue

If invoked with an issue number (e.g. `/python-tdd 42`) or path to local `.scratch/` file with issue description:
- Read the issue from the configured issue tracker (GitHub Issues if configured, or local `.scratch/` files).
- If no tracker is configured, ask the user to describe the issue inline.

If invoked without an argument, ask the user to describe what needs to be implemented.

---

## Step 2 — Load references

Read **all** of the following reference files from this skill's own `references/` directory before writing any code. Read them in order — `references/rules-digest.md` first:

1. `references/rules-digest.md`
2. `references/python-style.md`
3. `references/advanced-typing.md`
4. `references/visibility.md`
5. `references/idioms.md`
6. `references/comments.md`
7. `references/magic-methods.md`
8. `references/file-organization.md`
9. `references/project-structure.md`
10. `references/testing.md`
11. `references/debugging.md`
12. `references/error-handling.md`
13. `references/data-structures.md`
14. `references/class-design.md`
15. `references/pydantic.md`
16. `references/configuration.md`
17. `references/pathlib.md`
18. `references/concurrency.md`
19. `references/async.md`
20. `references/logging.md`
21. `references/security.md`
22. `references/numpy-types.md`
23. `references/plotting.md`

Do not skip any file. Do not try to infer which ones apply — load all of them.

---

## Step 3 — Plan

Before writing any code:

- [ ] Check `CONTEXT.md` for domain vocabulary — use its terms in test names and interface design
- [ ] Check `docs/adr/` for any decisions that constrain the area you're touching
- [ ] Infer needed interface changes from the issue/spec, existing code, tests, `CONTEXT.md`, and ADRs
- [ ] When designing new interfaces: prefer returning results over side effects, and accept dependencies rather than creating them internally
- [ ] Infer which behaviours to test, prioritising the critical path
- [ ] List the behaviours as a numbered sequence — observable behaviours, not implementation steps
- [ ] Proceed without user approval unless there is a blocking ambiguity that would make implementation unsafe

Ask only when the issue/spec/codebase is insufficient to proceed safely. Otherwise, write down the inferred plan and continue.

---

## Step 4 — Tracer bullet

Write ONE test that confirms ONE thing about the system:

```
RED:   Write test for first behaviour → run `uv run poe check` → confirm it fails
GREEN: Write minimal code to pass → run `uv run poe check` → confirm it passes
```

This is your tracer bullet — it proves the end-to-end path works.

---

## Step 5 — Incremental loop

**Anti-pattern: Horizontal Slices.** Do NOT write all tests first, then all implementation. This produces tests that verify imagined behaviour rather than actual behaviour, and you outrun your headlights before understanding the implementation.

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
  ...
```

For each remaining behaviour:

```
RED:   Write next test → `uv run poe check` → confirm failure
GREEN: Minimal code to pass → `uv run poe check` → confirm pass
```

Rules:
- One test at a time
- Only enough code to pass the current test
- Do not anticipate future tests
- Keep tests focused on observable behaviour through public interfaces
- **Never refactor while RED**

When `poe check` stops on a failure:
1. Read the reported error
2. Fix that one thing (a lint violation `--fix` could not auto-resolve, a type error, or a failing test)
3. Run `uv run poe check` again
4. Repeat until it passes cleanly

Do not use `# type: ignore` to silence a type error unless it is a known mypy limitation — document why with a comment on the same line. If a test fails that you did not write, investigate before assuming it is pre-existing — you may have introduced a regression. Coverage must stay at or above 80%.

**Per-cycle checklist:**
```
[ ] Test describes behaviour, not implementation
[ ] Test uses public interface only
[ ] Test would survive an internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
```

---

## Step 6 — Post-implementation review gate

After all tests pass, always run a size-scaled review of the uncommitted diff before reporting done.

Use git only for read-only inspection:

- [ ] Inspect `git diff`
- [ ] Inspect `git diff --staged` if staged changes exist
- [ ] Do not stage, unstage, commit, reset, checkout, switch branches, or otherwise change git state

Check the whole uncommitted diff for:

- **Repo-documented standards** — find and follow any local standards such as `CODING_STANDARDS.md`, `CONTRIBUTING.md`, or equivalent. Documented repo standards override the smell baseline.
- **Scope creep** — remove behaviour, abstractions, parameters, or hooks not requested by the issue/spec.
- **Mysterious Name** — a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code** — the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy** — a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps** — the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession** — a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches** — the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery** — one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change** — one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality** — abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains** — long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man** — a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest** — a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

Resolution rules:

- Fix hard blockers before completion: failing `uv run poe check`, repo-standard violations, implementation that contradicts requested behaviour, and clear unintended scope creep.
- Automatically fix judgement-call smells when the refactor is obvious, local, and low-risk.
- Leave broad, risky, or scope-expanding judgement calls as follow-up work in the final report instead of stopping for user input.
- Do not optimise code unless you have measured that it is a bottleneck.
- Consider what the new code reveals about existing code — refactor existing code too if needed and still within scope.
- Run `uv run poe check` after each fix or refactor step.

---

## Step 7 — Report

Tell the user:
- What you changed and why
- Which tests cover the change (new or existing)
- Any trade-offs or follow-up work needed
- What the post-implementation review gate found, including any broad/risky judgement calls left as follow-up

---

## Done criterion

**The task is not done until `uv run poe check` passes cleanly** — no lint errors, no type errors, no failing tests, coverage at or above 80%.
