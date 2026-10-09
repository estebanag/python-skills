---
name: setup-python-quality
description: Set up a Python project's quality tooling and `uv run poe check` gate. Run once per project before using python-quality.
---

# Setup Python Quality

Bootstraps a Python project's quality tooling by merging the required config into `pyproject.toml`.

---

## Step 1 — Check pyproject.toml

Ask for the importable Python package name (snake_case, e.g. `my_library`) to fill in
`<package_name>` before checking project files.

1. Read `pyproject.toml` and `assets/pyproject.standard.toml` in full; substitute `<package_name>` in the asset with the importable package name in memory.
2. Compare all standard sections and keys with the project, including `tool.poe.tasks.check`, its referenced tasks, and the dependency groups needed to run them. Check whether `src/<package_name>/py.typed` exists when the package directory exists. Section presence alone does not establish that the gate is configured.
3. List missing keys and any existing keys with different values side by side. Confirm additions with the user; for differing values, ask which to keep. Respect the project's existing configuration rather than silently replacing it.
4. On confirmation, merge missing keys into `pyproject.toml`. Change an existing value only if the user explicitly chooses the standard value. If `src/<package_name>/` exists but lacks `py.typed`, ask before adding the empty marker.
5. Recheck that `uv run poe check` is defined and that its referenced tasks and required tool dependencies are configured. If the user keeps a configuration that prevents the gate from running, report the blocker rather than declaring setup complete. If the package directory does not exist yet, remind the user to add `src/<package_name>/py.typed` when it is created.

If nothing needs changing and the gate is configured, tell the user the project is already configured.

---

## Step 2 — Sync dependencies

After making changes, report what was added or updated, including any `py.typed` marker action. Derive the appropriate `uv sync` command from the project's actual dependency groups and ask the user to run it before continuing (for the standard asset's `dev` and `test` groups: `uv sync --group dev --group test`).
