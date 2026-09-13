---
name: setup-python-tdd
description: Bootstrap a Python project for TDD by updating pyproject.toml. Run once per project before using python-tdd.
---

# Setup Python TDD

Bootstraps a Python project with standard TDD tooling by merging the required config into `pyproject.toml`.

---

## Step 1 — Check pyproject.toml

Ask for the importable Python package name (snake_case, e.g. `my_library`) to fill in
`<package_name>` before checking project files.

Check whether `[tool.poe.tasks]`, `[tool.mypy]`, and `[tool.ruff]` all exist in `pyproject.toml`.
Also check whether `src/<package_name>/py.typed` exists when `src/<package_name>/` exists.

**If all three tool sections exist and `src/<package_name>/py.typed` exists:** tell the user the project is already configured and stop.

**If all three tool sections exist but `src/<package_name>/` does not exist yet:** tell the user the tooling is configured, and remind them to add `src/<package_name>/py.typed` when the package directory is created.

**If any tool section is missing or `py.typed` is missing from an existing package directory:**

1. Read `pyproject.toml` in full.
2. Read `assets/pyproject.standard.toml` in full and substitute `<package_name>` with the importable Python package name in memory.
3. Compare `pyproject.toml` against the standard sections from the asset. Identify every key that is **absent** from the existing file.
4. Show the user the list of missing sections and ask for confirmation before making any change:
   > "The following standard sections are missing from `pyproject.toml`: [list]. May I add them?"
5. If any key already exists with a **different value** from the standard, show both values side by side and ask the user which to keep before proceeding.
6. On confirmation, **merge** the missing keys into `pyproject.toml` — never remove or overwrite any existing key.
7. If `src/<package_name>/` exists and `src/<package_name>/py.typed` is missing, ask for confirmation and add an empty `py.typed` marker file.
8. If `src/<package_name>/` does not exist yet, include a reminder in the final report to add `src/<package_name>/py.typed` when the package directory is created.

---

## Step 2 — Sync dependencies

After making changes, tell the user what was added or updated, including any `py.typed`
marker action, and ask them to run `uv sync --group dev --group test` before continuing.
