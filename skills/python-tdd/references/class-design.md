# Class Design

Conventions for designing Python classes. Read this before adding a new class.

## `@classmethod`: alternative constructors

Use `@classmethod` to provide alternative construction paths when a single `__init__` signature would become awkward.

```python
from __future__ import annotations

import json
from dataclasses import dataclass
from pathlib import Path
from typing import Any

@dataclass
class Config:
    """Application configuration."""
    host: str
    port: int
    debug: bool = False

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> Config:
        """Construct a Config from a raw dictionary."""
        return cls(
            host=str(data["host"]),
            port=int(data["port"]),
            debug=bool(data.get("debug", False)),
        )

    @classmethod
    def from_json_file(cls, path: Path) -> Config:
        """Load a Config from a JSON file."""
        return cls.from_dict(json.loads(path.read_text(encoding="utf-8")))
```

**Rules:**
- Name factory classmethods `from_<source>` — `from_dict`, `from_file`, `from_env`.
- Prefer `@classmethod` factories over overloading `__init__` with many optional parameters.
- Always annotate the return type as the class itself (use `cls` return for inheritance safety when needed).

## `@staticmethod` vs module-level function

`@staticmethod` is a function that lives in the class namespace but receives neither `self` nor `cls`.

**Use `@staticmethod`** when:
- The function logically belongs to the class (it operates on the class's domain)
- But it needs neither instance state nor class state

**Use a module-level function** when:
- The function is a utility with no conceptual tie to the class
- Or when you want callers to be able to import it directly

```python
class Invoice:
    """Represents a sales invoice."""

    @staticmethod
    def _format_currency(amount: float, symbol: str = "£") -> str:
        """Format a float as a currency string."""
        return f"{symbol}{amount:.2f}"

    def total_display(self) -> str:
        return self._format_currency(self.total)
```

```python
# ❌ @staticmethod for a general-purpose utility — use a module function
class StringUtils:
    @staticmethod
    def slugify(text: str) -> str: ...  # no connection to StringUtils

# ✅ module-level function
def slugify(text: str) -> str: ...
```

## `@property`: computed attributes

Use `@property` for attributes that are computed from instance state, have no side effects, and feel like attribute access to callers.

```python
from dataclasses import dataclass

@dataclass
class Rectangle:
    width: float
    height: float

    @property
    def area(self) -> float:
        """Return the area of the rectangle."""
        return self.width * self.height

    @property
    def perimeter(self) -> float:
        """Return the perimeter of the rectangle."""
        return 2 * (self.width + self.height)
```

**Rules:**
- Properties must be cheap to compute — callers assume `obj.area` is O(1).
- For expensive computations, use `@functools.cached_property` instead (evaluated once, cached on the instance).
- Avoid `@property` setters unless the class has a strong reason to validate on assignment — plain dataclass fields are simpler.
- Never raise unexpected exceptions from a property — callers don't expect attribute access to fail.

```python
from functools import cached_property

class Document:
    def __init__(self, path: Path) -> None:
        self._path = path

    @cached_property
    def content(self) -> str:
        """Read and cache the file content."""
        return self._path.read_text(encoding="utf-8")  # expensive — read once
```

## `__slots__`: memory-efficient classes

`__slots__` replaces the per-instance `__dict__` with a fixed set of attributes, reducing memory usage significantly for classes with many instances.

### Preferred: `@dataclass(slots=True)` (Python 3.10+)

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    """A 2D point — slots reduces memory for large collections of points."""
    x: float
    y: float
```

`@dataclass(slots=True)` generates `__slots__` automatically. Use it when a dataclass will be instantiated in large numbers or memory use matters.

### Manual `__slots__` for non-dataclass classes

```python
class Node:
    """A linked-list node."""
    __slots__ = ("value", "next")

    def __init__(self, value: int) -> None:
        self.value = value
        self.next: Node | None = None
```

**Trade-offs:**
- Classes with `__slots__` have no `__dict__` — you cannot add arbitrary attributes at runtime.
- `__slots__` classes cannot be used with `weakref` unless `"__weakref__"` is in `__slots__`.
- Inheritance with `__slots__` requires each class in the hierarchy to define its own `__slots__`.

**Rule:** add `slots=True` when many instances are expected or memory matters. Omit it when you need dynamic attribute assignment or weakref support.
