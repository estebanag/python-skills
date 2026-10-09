# Debugging

How to investigate failures in this project.

## `breakpoint()` / pdb

Drop into the interactive debugger at any line:

```python
def process(data: list[str]) -> list[str]:
    breakpoint()  # execution pauses here; opens pdb
    return sorted(data)
```

Key pdb commands:

| Command | Action |
|---------|--------|
| `n` | Execute next line (step over) |
| `s` | Step into function call |
| `c` | Continue until next breakpoint |
| `p expr` | Print the value of `expr` |
| `pp expr` | Pretty-print `expr` |
| `l` | List source around current line |
| `u` / `d` | Move up/down the call stack |
| `q` | Quit debugger |

**Remove all `breakpoint()` calls before committing.**

## Pytest + pdb

Drop into the debugger automatically on test failures:

```bash
uv run pytest --pdb  # enter pdb on first failure
uv run pytest --pdb --pdbcls=IPython.core.debugger:Pdb  # use IPython if available
```

## Assertions for invariants

Use `assert` to document and enforce invariants — conditions that must always be true:

```python
def binary_search(items: list[int], target: int) -> int:
    assert items == sorted(items), "binary_search requires a sorted list"
    ...
```

**Rules:**
- Use `assert` for programmer errors and invariants — not for input validation from untrusted sources (assertions can be disabled with `-O`).
- For input validation, raise `ValueError` or a domain-specific exception instead.

## Structured print debugging

When `breakpoint()` is not enough and you need to trace a sequence of values, use explicit labelled prints — not bare `print(x)`:

```python
# ✅ labelled, easy to grep and remove
print(f"DEBUG process input: {data!r}")
result = transform(data)
print(f"DEBUG process output: {result!r}")

# ❌ unlabelled — impossible to find and remove later
print(data)
print(result)
```

**Always remove debug prints before committing.**

## Logging for persistent instrumentation

For instrumentation you want to keep in production, use `logging.debug()` rather than `print()`:

```python
import logging

logger = logging.getLogger(__name__)

def process(data: list[str]) -> list[str]:
    logger.debug("process called with %d items", len(data))
    result = sorted(data)
    logger.debug("process returning %d items", len(result))
    return result
```

See `logging.md` for the full logging conventions.

## Reading tracebacks

Python tracebacks are bottom-up — the **last frame** is where the error occurred. Read from the bottom up to find the root cause. The `--tb=short` pytest flag gives condensed tracebacks without full context.

## Common mistakes

```python
# ❌ catching all exceptions silently — hides bugs
try:
    result = compute()
except Exception:
    pass

# ✅ log the exception and re-raise, or handle specifically
try:
    result = compute()
except ValueError as exc:
    logger.exception("compute failed with invalid value")
    raise
```
