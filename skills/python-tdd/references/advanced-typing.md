# Advanced Typing

Patterns for complex type annotations under strict mypy. Read this before writing generic code, protocols, or overloaded functions.

## Abstract types for function parameters

Annotate parameters with abstract types from `collections.abc` so callers can
pass any compatible container; annotate returns with the concrete type so
callers know exactly what they get.

```python
from collections.abc import Mapping
from collections.abc import Sequence

def merge_labels(
    labels: Mapping[str, str], extra: Sequence[str]
) -> dict[str, str]:
  ...
```

## Generic functions (PEP 695 type parameters)

Use the native type-parameter syntax (Python 3.12+) to write functions that preserve the type of their argument — no `TypeVar` import or module-level declaration needed.

```python
# ✅ PEP 695 — inline type parameter, no import
from __future__ import annotations

def first[T](items: list[T]) -> T:
    """Return the first element of the list."""
    return items[0]

# mypy knows: first(["a", "b"]) -> str
# mypy knows: first([1, 2]) -> int

# ❌ legacy — module-level TypeVar with an explicit import
from typing import TypeVar

T = TypeVar("T")

def first_legacy(items: list[T]) -> T:
    return items[0]
```

**Rules:**
- Prefer the inline `[T]` syntax over the legacy `T = TypeVar("T")` form — it scopes the parameter to the function or class that uses it and needs no import.
- Name it `T`, `S`, `K`, `V` for generic use; use descriptive names (`NodeT`, `ItemT`) when the bound matters.
- Add a bound inline: `def smallest[T: Comparable](items: list[T]) -> T`.
- The legacy `typing.TypeVar` form is still valid and occasionally needed (e.g. explicit variance, `TypeVar("T", covariant=True)`, which is not yet expressible inline) — reach for it only then.

## Generic classes (PEP 695)

Make a class generic with the inline `[T]` syntax — no `Generic[T]` base or `TypeVar` needed.

```python
# ✅ PEP 695
from __future__ import annotations

class Stack[T]:
    """A type-safe stack."""

    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        """Push an item onto the stack."""
        self._items.append(item)

    def pop(self) -> T:
        """Pop and return the top item."""
        return self._items.pop()

# ❌ legacy — Generic[T] base plus a module-level TypeVar
from typing import Generic, TypeVar

T = TypeVar("T")

class StackLegacy(Generic[T]): ...
```

## Type aliases (PEP 695 `type` statement)

Declare aliases with the `type` statement (Python 3.12+), not the legacy `TypeAlias` annotation.

```python
# ✅ PEP 695 — type statement (supports its own parameters)
type Json = dict[str, "Json"] | list["Json"] | str | int | float | bool | None
type Matrix[T] = list[list[T]]

# ❌ legacy
from typing import TypeAlias

Vector: TypeAlias = list[float]
```

## Protocol

Use `Protocol` for structural subtyping (duck typing with type safety). Prefer it over ABCs for interfaces that third-party code should satisfy without inheriting.

```python
from __future__ import annotations
from typing import Protocol

class Serialisable(Protocol):
    """Any object that can serialise itself to a dict."""

    def to_dict(self) -> dict[str, object]: ...

def save(obj: Serialisable) -> None:
    """Save any serialisable object."""
    data = obj.to_dict()
    ...
```

**Rules:**
- Protocol methods have `...` bodies (or `pass`) — they are structural contracts, not implementations.
- Use `runtime_checkable` only when you need `isinstance` checks.
- Protocols live in a `_protocols.py` or `_interfaces.py` module to avoid circular imports.

## TypeGuard

Use `TypeGuard` for type-narrowing functions that inspect a value at runtime.

```python
from __future__ import annotations
from typing import TypeGuard

def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    """Return True if every element is a str."""
    return all(isinstance(x, str) for x in val)

def first_upper(items: list[object]) -> str | None:
    """Return the first item uppercased if the input is a string list."""
    if is_string_list(items):
        # mypy knows items: list[str] here
        return items[0].upper()
    return None
```

## @overload

Use `@overload` when a function has different return types depending on argument types.

```python
from __future__ import annotations
from typing import overload

@overload
def parse(value: str) -> int: ...
@overload
def parse(value: bytes) -> float: ...

def parse(value: str | bytes) -> int | float:
    """Parse a string to int or bytes to float."""
    if isinstance(value, str):
        return int(value)
    return float(value)
```

**Rules:**
- All `@overload` stubs come before the actual implementation.
- The implementation signature uses the union of all overload types.
- Never call the implementation directly when `@overload` stubs exist.

## ParamSpec

Use the inline `[**P, R]` type parameters (PEP 695) to type decorators that preserve the wrapped function's signature — no `ParamSpec`/`TypeVar` import needed.

```python
# ✅ PEP 695 — inline type parameters
from __future__ import annotations

import logging
from collections.abc import Callable
from functools import wraps

logger = logging.getLogger(__name__)

def log_call[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    """Log every call to the wrapped function."""
    @wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        logger.debug("calling %s", fn.__name__)
        return fn(*args, **kwargs)
    return wrapper
```

The legacy form imports `ParamSpec` and `TypeVar` from `typing` and declares them at module level — prefer the inline syntax above.

## TYPE_CHECKING guard

Use `TYPE_CHECKING` to import types that are only needed for annotations, avoiding circular imports at runtime.

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from mypackage.heavy_module import HeavyClass

def process(obj: HeavyClass) -> None:  # annotation-only, no runtime import
    ...
```

**Rule:** Always combine with `from __future__ import annotations` so the annotation is never evaluated at runtime.
