# Visibility

Conventions for controlling what is public, internal, and private in this project.

## Naming conventions

| Prefix | Meaning | Accessible from |
|--------|---------|-----------------|
| `name` | Public API | Anywhere |
| `_name` | Module-private / internal | Same module or package (by convention — not enforced) |
| `__name` | Name-mangled | Only the defining class (`_ClassName__name`) |

```python
class Processor:
    max_retries: int = 3  # public class variable
    _registry: dict[str, int] = {}  # internal, not part of public API
    __secret_key: str = "..."  # name-mangled, truly private

    def process(self, data: str) -> str: ...  # public method
    def _validate(self, data: str) -> bool: ...  # internal helper
    def __hash_key(self, key: str) -> int: ...  # name-mangled
```

**Rules:**
- Use `_` for anything that is an implementation detail — not meant for callers.
- Use `__` only for class internals where subclass name collisions are a real concern (rare).
- Never access `_name` members from outside the module except in tests.

## `__all__`: defining the public API

Every module that is part of the public API must define `__all__`. This explicitly declares what is exported and prevents `from module import *` from leaking internals.

```python
# mypackage/processor.py
from __future__ import annotations

__all__ = ["Processor", "ProcessorConfig"]

class Processor: ...
class ProcessorConfig: ...
class _InternalHelper: ...  # not in __all__, stays private
```

**Rules:**
- Define `__all__` near the top of the file, after imports.
- Only include names that external callers should use.
- Re-exporting in `__init__.py` must also appear in that file's `__all__`.

## `__init__.py`: public API surface

The package's `__init__.py` is the public contract. Import and re-export only what users should call.

```python
# mypackage/__init__.py
from __future__ import annotations

from mypackage.processor import Processor as Processor
from mypackage.processor import ProcessorConfig as ProcessorConfig
from mypackage.runner import run as run

__all__ = ["Processor", "ProcessorConfig", "run"]
```

**Rules:**
- Keep `__init__.py` thin — just re-exports, no logic.
- Do not import internal helpers or implementation modules here.
- Everything in `__all__` must be importable as `from mypackage import X`.

## `TYPE_CHECKING`: avoiding circular imports

When two modules need each other's types only for annotations, use the `TYPE_CHECKING` guard to break the cycle.

```python
# module_a.py
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from mypackage.module_b import ModuleB  # only imported during type checking

class ModuleA:
    def process(self, other: ModuleB) -> None: ...  # annotation is a string at runtime
```

**Rules:**
- Always combine `TYPE_CHECKING` imports with `from __future__ import annotations`.
- If a circular import exists at runtime (not just annotations), restructure: extract shared types into a third `_types.py` module.
- Never put `TYPE_CHECKING` imports outside the `if TYPE_CHECKING:` block.

## What callers should see

When writing a new module, ask:

1. What is the **minimum** public interface callers need?
2. Everything else is `_private`.
3. Register that minimum in `__all__`.
4. Re-export from `__init__.py` only if it belongs to the package's top-level API.

---

## Public API design

### Explicit re-exports in `__init__.py`

Re-export public symbols using the `as Name` form. This tells type checkers (and tools like `mypy --no-implicit-reexport`) that the symbol is intentionally part of the public API:

```python
# ✅ explicit re-export — mypy strict treats these as public
# mypackage/__init__.py
from mypackage._core import Processor as Processor
from mypackage._core import ProcessorConfig as ProcessorConfig
from mypackage._runner import run as run

__all__ = ["Processor", "ProcessorConfig", "run"]

# ❌ implicit import — under strict (--no-implicit-reexport) these are NOT
# considered public, so `from mypackage import Processor` fails type checking
from mypackage._core import Processor, ProcessorConfig
from mypackage._runner import run
```

Without `as Name`, mypy with `--no-implicit-reexport` (enabled under `strict`) will not consider the import a public re-export.

### Deprecation workflow

When a public symbol must be removed or replaced, always deprecate before removing:

```python
import warnings

def old_function(x: int) -> int:
    """Compute something.

    .. deprecated:: 2.1
        Use :func:`new_function` instead. Will be removed in 3.0.
    """
    warnings.warn(
        "old_function is deprecated and will be removed in 3.0. "
        "Use new_function instead.",
        DeprecationWarning,
        stacklevel=2,  # ✅ points at the caller, not this function (omitting it / stacklevel=1 ❌ blames the library)
    )
    return new_function(x)
```

**Rules:**
- Always set `stacklevel=2` — it makes the warning point at the caller's code, not at the library internals.
- Include the version when the symbol was deprecated and the version when it will be removed.
- Keep the deprecated symbol working for at least one minor release cycle before removal.

### Backward-compatibility rule

> **Never remove a public symbol without a deprecation cycle.**

Any name that appears in `__all__` or is documented in the public API is a contract. Breaking it is a breaking change requiring a major version bump (semver). The sequence is:

1. Deprecate in version N (add `warnings.warn`)
2. Remove in version N+1 (major) or after a stated timeline

Never silently change the behaviour of a public function. If the signature must change, add a new function and deprecate the old one.
