# Project Structure

Canonical layout and module boundary rules for this project.

## Directory layout

```
<project-root>/
├── src/
│   └── <package_name>/      ← all library source code lives here
│       ├── __init__.py      ← public API surface (re-exports only)
│       ├── py.typed         ← PEP 561 marker: this package ships inline types
│       └── ...              ← modules and sub-packages
├── tests/                   ← pytest tests (flat layout)
│   ├── conftest.py          ← shared fixtures
│   └── test_<module>.py
├── docs/
│   └── adr/                 ← architecture decision records
├── pyproject.toml           ← project metadata, deps, tool config
├── uv.lock                  ← committed lockfile
├── .python-version          ← pinned Python version for uv
├── AGENTS.md                ← AI context file (root-level)
└── README.md
```

## Where to put new code

| What you're adding | Where it goes |
|--------------------|--------------|
| A new public class or function | `src/<package>/<module>.py` |
| A private helper used by one module | Same file as its caller, prefixed `_` |
| A private helper used by multiple modules | `src/<package>/_helpers.py` (or a dedicated `_utils.py`) |
| Shared exception types | `src/<package>/exceptions.py` |
| Shared type aliases and Protocols | `src/<package>/_types.py` |
| A new test | `tests/test_<module>_<feature>.py` |
| A shared test fixture | `tests/conftest.py` |

## `__init__.py`: public API surface

`src/<package>/__init__.py` is the package's contract with the outside world. It should contain **only re-exports** — no logic.

```python
# src/mypackage/__init__.py
from __future__ import annotations

from mypackage.exceptions import MyPackageError as MyPackageError
from mypackage.exceptions import ValidationError as ValidationError
from mypackage.processor import Processor as Processor
from mypackage.processor import ProcessorConfig as ProcessorConfig
from mypackage.runner import run as run

__all__ = [
    "Processor",
    "ProcessorConfig",
    "run",
    "MyPackageError",
    "ValidationError",
]
```

**Rules:**
- If it's in `__all__`, callers can do `from mypackage import X`.
- If it's not in `__all__`, it's an implementation detail — do not document or rely on it.
- Do not put business logic, side effects, or imports of heavy dependencies here.

## Module boundaries

A module has one cohesive responsibility. Signs a module needs splitting:

- **Over ~300 lines** with multiple unrelated responsibilities.
- **Tests for one part don't need the other part.**
- **Distinct concerns** (e.g., parsing + validation + serialisation in one file).

When splitting `processor.py` into a sub-package:

```
src/mypackage/processor/
├── __init__.py      ← re-exports Processor, ProcessorConfig
├── _parser.py
├── _validator.py
└── _serialiser.py
```

The `__init__.py` keeps the external API unchanged:

```python
from mypackage.processor._parser import parse as parse
from mypackage.processor._serialiser import Processor as Processor
from mypackage.processor._serialiser import ProcessorConfig as ProcessorConfig
from mypackage.processor._validator import validate as validate

__all__ = ["Processor", "ProcessorConfig", "parse", "validate"]
```

## Avoiding circular imports

**Symptom:** `ImportError: cannot import name X from partially initialized module Y`.

**Strategy 1:** Use `TYPE_CHECKING` for annotation-only imports.

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from mypackage.other import OtherClass
```

**Strategy 2:** Extract shared types into `_types.py`.

If `module_a` and `module_b` both need `CommonType`, move it to `_types.py`. Both import from there; neither imports from the other.

**Strategy 3:** Late import inside a function (last resort).

```python
def get_processor() -> Processor:
    from mypackage.processor import Processor  # imported only when called
    return Processor()
```

## Deep modules

Prefer **deep modules**: a small, stable public interface that hides a complex implementation. Avoid shallow modules that expose every internal detail.

```python
# ✅ deep: simple interface hides complexity
class DataPipeline:
    def run(self, source: Path) -> list[Record]: ...

# ❌ shallow: callers must orchestrate internals themselves
class DataPipeline:
    def load(self, source: Path) -> RawData: ...
    def validate(self, data: RawData) -> ValidatedData: ...
    def transform(self, data: ValidatedData) -> list[Record]: ...
    def filter_duplicates(self, records: list[Record]) -> list[Record]: ...
```
