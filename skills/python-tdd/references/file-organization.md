# File Organization

Conventions for how source files and classes are structured internally.

## Module-level ordering

Every Python source file follows this top-to-bottom order:

1. Module docstring
2. `from __future__ import annotations`
3. Standard library imports
4. Third-party imports
5. Local imports
6. `__all__` (if public module)
7. Module-level constants (`UPPER_SNAKE_CASE`)
8. Exceptions defined in this module
9. Class definitions
10. Module-level functions
11. `if __name__ == "__main__":` block (scripts only)

```python
"""Utilities for processing data records."""

from __future__ import annotations

import json
from pathlib import Path

import numpy as np

from mypackage.core import BaseProcessor

__all__ = ["RecordProcessor"]

MAX_BATCH_SIZE = 500

class RecordProcessorError(Exception): ...

class RecordProcessor(BaseProcessor): ...

def _load_defaults() -> dict[str, object]: ...
```

## Class member ordering

Inside a class, members follow this order:

1. Class-level variables (type annotations and defaults)
2. `__init__`
3. `__post_init__` (dataclasses)
4. Class methods (`@classmethod`) and static methods (`@staticmethod`)
5. Properties (`@property` and their setters)
6. Public methods (alphabetical within a group is fine)
7. Private / internal methods (prefixed `_`)
8. Magic methods (`__repr__`, `__eq__`, etc.) — at the end

```python
class Processor:
    # 1. Class variables
    MAX_RETRIES: int = 3
    _registry: dict[str, Processor] = {}

    # 2. __init__
    def __init__(self, name: str) -> None: ...

    # 4. Class methods
    @classmethod
    def from_config(cls, config: dict[str, object]) -> Processor: ...

    # 5. Properties
    @property
    def name(self) -> str: ...

    # 6. Public methods
    def process(self, data: str) -> str: ...
    def validate(self, data: str) -> bool: ...

    # 7. Private helpers
    def _prepare(self, data: str) -> str: ...

    # 8. Magic methods
    def __repr__(self) -> str: ...
    def __eq__(self, other: object) -> bool: ...
```

## When to split a module into a package

Split a module `mypackage/processor.py` into `mypackage/processor/` when **any** of these are true:

- The file exceeds ~300 lines.
- It has multiple distinct responsibilities (e.g., parsing + validation + serialisation).
- Tests for one part don't need the other.

After splitting:

```
mypackage/processor/
├── __init__.py      ← re-exports the public API
├── _parser.py       ← internal
├── _validator.py    ← internal
└── _serialiser.py   ← internal
```

The `__init__.py` keeps the public interface unchanged:

```python
from mypackage.processor._parser import parse as parse
from mypackage.processor._validator import validate as validate

__all__ = ["parse", "validate"]
```

## File naming

- Module files: `snake_case.py`
- Private modules: `_snake_case.py`
- Test files: `test_<module>_<feature>.py` (see `testing.md`)
- Do not use hyphens in module names — `my-module.py` cannot be imported.

```python
# ✅ importable
import data_loader
from mypackage import _internal_cache

# ❌ not importable — hyphen is invalid in a module name
import data-loader  # SyntaxError
```

## One class per file?

No hard rule, but prefer one primary public class per file, with private helpers in the same file. Move to multiple files only when a file becomes hard to navigate.
