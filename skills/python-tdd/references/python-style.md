# Python Style

Coding conventions for this project. Read this before writing any Python code.

## Formatter: ruff

This project uses `ruff` for both formatting and linting.

```bash
uv run ruff format .          # format all files
uv run ruff check .           # lint
uv run ruff check --fix .     # auto-fix safe issues
```

Configuration (from `pyproject.toml`):
- **Line length:** 88 characters
- **Indentation:** 4 spaces
- **Quote style:** double quotes
- **Target:** Python 3.12+
- **Docstring convention:** Google style

## Type Annotations

All code must be fully type-annotated. This project enforces **strict mypy**.

```bash
uv run mypy src tests
```

**Rules:**
- Every function and method must annotate all parameters and the return type.
- Add `from __future__ import annotations` at the top of every module (enables deferred annotation evaluation, avoids forward-reference strings).
- Prefer precise types. `Any` is allowed as a deliberate escape valve for genuinely untyped or dynamic data (e.g. `dict[str, Any]` for arbitrary JSON) — but do not reach for it to silence a type error you could fix. Don't replace a needed `Any` with `object` or a bounded `TypeVar` just to avoid the word; use those only when they actually model the value.
- Use `X | Y` union syntax — not `Union[X, Y]`.
- Use `list[X]`, `dict[K, V]`, `tuple[X, ...]` (lowercase builtins) — not `List`, `Dict`, `Tuple` from `typing`.
- Use `X | None` — not `Optional[X]`.
- Use the `type` statement for type aliases (PEP 695): `type MyType = dict[str, int]` — not the legacy `MyType: TypeAlias = ...` form.
- Prefer PEP 695 inline type parameters for generics: `def first[T](items: list[T]) -> T` and `class Stack[T]` — not module-level `TypeVar` + `Generic[T]`. See `advanced-typing.md`.

```python
# ✅ correct
from __future__ import annotations

def process(items: list[str], limit: int = 10) -> list[str]: ...

# ❌ wrong
from typing import List, Optional
def process(items: List[str], limit: Optional[int] = 10) -> List[str]: ...
```

## Docstrings

All **public** symbols must have docstrings. Use **Google style**.

```python
def fetch_records(source: str, max_count: int = 100) -> list[dict[str, str]]:
    """Fetch records from the given source.

    Args:
        source: The data source identifier.
        max_count: Maximum number of records to return. Defaults to 100.

    Returns:
        A list of record dicts, each with string keys and values.

    Raises:
        ValueError: If source is empty.
        ConnectionError: If the source is unreachable.
    """
```

**Rules:**
- First line: one-sentence imperative summary ("Fetch records", not "Fetches records").
- Include `Args:`, `Returns:`, `Raises:` sections only when they add information beyond the type annotations.
- Module docstring: one sentence describing what the module contains.
- Class docstring: describes the class purpose. Document `__init__` params in `__init__`, not the class docstring.
- Private symbols (prefixed `_`) do not require docstrings.

## Imports

Managed by ruff's isort. Order:

1. Standard library
2. Third-party packages
3. Local package imports

Separated by blank lines. No wildcard imports (`from x import *`). No implicit relative imports.

```python
# ✅ correct
from __future__ import annotations

import os
from pathlib import Path

import numpy as np

from mypackage.core import BaseProcessor
```

## General Conventions

- **No mutable default arguments.** Use `None` as default, initialise inside the function.
- **f-strings** over `.format()` or `%` for string interpolation.
- **`UPPER_SNAKE_CASE`** for module-level constants.
- **`_snake_case`** prefix for private helpers.
- **`dataclasses.dataclass`** over plain dicts for structured data by default; use
  `pydantic.BaseModel` when validation, schema, or serialisation is needed and pydantic is
  available.
- **Explicit `raise X from Y`** when re-raising inside an except block.

```python
# ✅ no mutable defaults
def collect(items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    return items

# ❌ mutable default
def collect(items: list[str] = []) -> list[str]: ...
```

## Strings and bytes

### Always specify `encoding` in `open()`

The default encoding is platform-dependent (`locale.getpreferredencoding()`). On Windows it may be `cp1252`; on Linux, `utf-8`. Never rely on it:

```python
# ✅
with open("data.txt", encoding="utf-8") as f:
    content = f.read()

# ✅ same with pathlib (preferred)
content = Path("data.txt").read_text(encoding="utf-8")

# ❌ platform-dependent — breaks on Windows
with open("data.txt") as f:
    content = f.read()
```

### Use f-strings for interpolation

```python
# ✅
message = f"Processing {count} records from {source!r}"

# ❌ .format()
message = "Processing {} records from {!r}".format(count, source)

# ❌ % formatting (except in logging — see logging.md for why)
message = "Processing %d records from %r" % (count, source)
```

**Exception:** in `logging` calls, use `%s` lazy formatting — the logging module defers string construction until the message is actually emitted. See `logging.md`.

### Explicit `.encode()` / `.decode()` at I/O boundaries

Never mix `str` and `bytes` implicitly. Convert explicitly at the system boundary (where you read from or write to external storage), not deep inside business logic:

```python
# ✅ encode at the write boundary
def write_payload(path: Path, data: str) -> None:
    path.write_bytes(data.encode("utf-8"))

# ✅ decode at the read boundary
def read_payload(path: Path) -> str:
    return path.read_bytes().decode("utf-8")

# ❌ passing bytes where str is expected (or vice versa)
def process(data: str) -> str:
    return data.upper()

raw: bytes = b"hello"
process(raw)  # TypeError at runtime — mypy catches this with strict mode
```
