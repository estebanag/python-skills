# Rules digest — read this first

The non-negotiable rules below are **enforced mechanically** by `ruff`, `mypy --strict`, and
`pytest`. Code that breaks them fails `uv run poe check` and is not done. The detailed prose
references that follow this digest explain the *why* — but this checklist is the floor.

## Always

- Run `uv run poe check` until it passes cleanly — that is the definition of done.
- Start every module with `from __future__ import annotations` (auto-inserted by ruff isort).
- Fully annotate every function/method: all parameters and the return type.
- Use modern typing: `X | Y`, `X | None`, `list[X]`, `dict[K, V]` — never `Union`, `Optional`, `List`, `Dict`.
- Use PEP 695 (3.12+): `def f[T](...)`, `class C[T]`, `type Alias = ...` — not module-level `TypeVar`/`Generic[T]`/`TypeAlias`.
- Use `pathlib.Path`, never `os.path` (enforced by `PTH`).
- Use the `logging` module, never `print()` (enforced by `T20`).
- Use lazy `%s` logging args, not f-strings, inside logging calls.
- Pass `encoding="utf-8"` to every `open()` / `read_text()` / `write_text()`.
- Google-style docstrings on every public symbol (enforced by `D`).
- `raise X from Y` when re-raising inside an `except` block.

## Never

- No `Any` to silence a fixable type error. `Any` is allowed *only* as a deliberate escape valve
  for genuinely untyped/dynamic data (e.g. `dict[str, Any]` for arbitrary JSON).
- No blanket `# type: ignore` — use `# type: ignore[code]` with a reason, and only for real mypy
  limitations (enforced by `PGH003`).
- No bare `except:` and no mutable default arguments (`def f(x: list = [])`).
- No `os.system`, `eval`, `exec`, `shell=True`, or `pickle` on untrusted data (enforced by `S`).
- No committed secrets — load config from env / `.env` (see `configuration`).

## Gates

- `pytest` must pass; coverage must stay ≥ 80%.
- Warnings are errors (`filterwarnings = error`) — fix the cause, don't suppress.

When a check fails: read the one reported error, fix that single thing, re-run `uv run poe check`,
repeat. Do not batch-guess fixes.
