# Magic Methods

When to implement Python's special (dunder) methods, and when to leave them out.

## `__repr__`

**Always implement** for any non-trivial class. It is the developer-facing string representation, used in the REPL, debuggers, and error messages.

```python
@dataclass
class Point:
    x: float
    y: float

    def __repr__(self) -> str:
        return f"Point(x={self.x!r}, y={self.y!r})"
```

**Rules:**
- Should produce a string that, when `eval()`-ed, recreates the object (or is close to it).
- Use `!r` for field values so strings are quoted.
- `@dataclass` generates `__repr__` automatically — no need to write it manually.

## `__str__`

Implement only when the **human-readable display** should differ from the developer repr.

```python
class Duration:
    def __init__(self, seconds: float) -> None:
        self.seconds = seconds

    def __repr__(self) -> str:
        return f"Duration(seconds={self.seconds!r})"

    def __str__(self) -> str:
        # Human-friendly display
        minutes, secs = divmod(int(self.seconds), 60)
        return f"{minutes}m {secs}s"
```

If `__str__` is not defined, Python falls back to `__repr__`.

## `__eq__` and `__hash__`

Implement `__eq__` when equality should be based on value, not identity.

**Critical rule:** If you define `__eq__`, you MUST also define `__hash__` — or explicitly set `__hash__ = None` (making the object unhashable).

```python
class Vector:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector):
            return NotImplemented  # ✅ lets Python try the reflected operation
        return self.x == other.x and self.y == other.y

    def __hash__(self) -> int:
        return hash((self.x, self.y))
```

```python
# ❌ returning False on a type mismatch — blocks the other operand's __eq__,
# so Vector(...) == SomeCompatibleType(...) can never be True
def __eq__(self, other: object) -> bool:
    if not isinstance(other, Vector):
        return False
    return self.x == other.x and self.y == other.y
```

```python
# Mutable objects that define __eq__ should be unhashable:
class MutableRecord:
    def __eq__(self, other: object) -> bool: ...

    __hash__ = None
```

If a type checker complains about assigning `None` to `__hash__`, prefer documenting that
limitation near the assignment rather than making the object hashable.

**Rules:**
- Return `NotImplemented` (not `False`) when the type doesn't match — lets Python try the reflected operation.
- `__hash__` must be consistent with `__eq__`: objects that compare equal must have the same hash.
- `@dataclass(eq=True)` generates `__eq__` automatically; `@dataclass(frozen=True)` also generates `__hash__`.

## `__len__`

Implement when the object has a natural notion of size.

```python
class Playlist:
    def __init__(self, tracks: list[str]) -> None:
        self._tracks = tracks

    def __len__(self) -> int:
        return len(self._tracks)

# Enables: len(playlist), bool(playlist)
```

Note: `bool(obj)` falls back to `__len__` if `__bool__` is not defined — an empty collection will be falsy.

## `__contains__`

Implement for membership testing (`in` operator).

```python
def __contains__(self, item: object) -> bool:
    return item in self._tracks
```

## `__iter__` and `__next__`

Implement `__iter__` to make an object iterable (usable in `for` loops, `list()`, etc.).

```python
from collections.abc import Iterator

class CountDown:
    def __init__(self, start: int) -> None:
        self._current = start

    def __iter__(self) -> Iterator[int]:
        return self

    def __next__(self) -> int:
        if self._current <= 0:
            raise StopIteration
        value = self._current
        self._current -= 1
        return value
```

Prefer `__iter__` returning a generator over implementing `__next__` directly:

```python
def __iter__(self) -> Iterator[int]:
    yield from range(self._current, 0, -1)
```

## `__enter__` and `__exit__`

Implement for objects that manage resources (used with `with`).

```python
class ManagedConnection:
    def __enter__(self) -> ManagedConnection:
        self._connect()
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> bool | None:
        self._disconnect()
        return None  # do not suppress exceptions
```

Prefer `contextlib.contextmanager` for simple cases (see `idioms.md`).

## When NOT to implement magic methods

- Do not implement `__del__` — it is called by the garbage collector at unpredictable times. Use context managers instead.
- Do not implement arithmetic operators (`__add__`, `__mul__`, etc.) unless the class genuinely represents a mathematical object.
- Do not implement `__getattr__` / `__setattr__` to store data — use regular attributes or `@property`.
