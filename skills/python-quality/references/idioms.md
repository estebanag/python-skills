# Python Idioms

Idiomatic Python patterns used in this project. Prefer these over verbose alternatives.

## Comprehensions

Use comprehensions for simple transformations and filters. Fall back to a loop when the logic is complex or has side effects.

```python
# ✅ list comprehension
names = [user.name for user in users if user.active]

# ✅ dict comprehension
scores = {user.id: user.score for user in users}

# ✅ set comprehension
unique_tags = {tag for post in posts for tag in post.tags}

# ❌ verbose loop for a simple transform
names = []
for user in users:
    if user.active:
        names.append(user.name)
```

**Rules:**
- One comprehension per line maximum. If it wraps, use a loop instead.
- No side effects inside comprehensions (no `print`, no mutation of external state).

## Generators

Use generators for lazy sequences that don't need to be materialised all at once.

```python
def read_chunks(path: Path, size: int = 4096) -> Generator[bytes, None, None]:
    """Yield successive chunks from a file."""
    with path.open("rb") as f:
        while chunk := f.read(size):
            yield chunk
```

Use `yield from` to delegate to a sub-generator:

```python
def all_items(sources: list[Iterable[str]]) -> Generator[str, None, None]:
    for source in sources:
        yield from source
```

## Context managers

Always use `with` for resources that need cleanup (files, locks, connections).

```python
# ✅
with path.open("r", encoding="utf-8") as f:
    content = f.read()

# ❌ manual close
f = path.open("r")
content = f.read()
f.close()
```

Write custom context managers with `contextlib.contextmanager`:

```python
import logging
import time
from contextlib import contextmanager
from collections.abc import Generator

logger = logging.getLogger(__name__)

@contextmanager
def timer(label: str) -> Generator[None, None, None]:
    """Log elapsed time for the block."""
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        logger.debug("%s: %.3fs", label, elapsed)
```

## Walrus operator (`:=`)

Use `:=` to assign and test in a single expression. Best in `while` loops and comprehensions.

```python
# ✅ reading until empty
while chunk := file.read(8192):
    process(chunk)

# ✅ avoiding double lookup in a comprehension
results = [y for x in data if (y := transform(x)) is not None]

# ❌ don't use walrus just to save a line — prefer clarity
```

## dataclasses

Use `@dataclass` for simple data holders with no validation logic.

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    """A 2D point."""
    x: float
    y: float

@dataclass
class Config:
    """Application configuration."""
    host: str = "localhost"
    port: int = 8080
    tags: list[str] = field(default_factory=list)  # mutable default via field()
```

**Rules:**
- Use `field(default_factory=...)` for mutable defaults (never `tags: list[str] = []`).
- Use `@dataclass(frozen=True)` for immutable value objects.
- Use `@dataclass(slots=True)` (Python 3.10+) when many instances are expected or memory matters.
- If you need validation, use `pydantic.BaseModel` instead (see `pydantic.md`).

## Unpacking

```python
# Swap without temp
a, b = b, a

# Star unpacking
first, *rest = items
*init, last = items

# Ignore values
_, important, *_ = record
```

## Common anti-patterns to avoid

```python
# ❌ checking type with == instead of isinstance
if type(x) == str: ...
# ✅
if isinstance(x, str): ...

# ❌ using exception for control flow
try:
    return d[key]
except KeyError:
    return default
# ✅
return d.get(key, default)

# ❌ range(len(...))
for i in range(len(items)):
    process(items[i])
# ✅
for item in items:
    process(item)
# or with index:
for i, item in enumerate(items):
    process_indexed(i, item)
```

## `functools` utilities

### `@functools.cache` and `@functools.lru_cache`

Cache the return value of a pure function so repeated calls with the same arguments are free:

```python
from functools import cache, lru_cache

@cache  # unbounded — equivalent to lru_cache(maxsize=None)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

@lru_cache(maxsize=128)  # bounded LRU — evicts least-recently-used entries
def parse_config(path: str) -> dict[str, str]:
    return _load_from_disk(path)
```

**GC gotcha:** `@lru_cache` on a method holds a strong reference to `self` through the cache key, preventing the instance from being garbage-collected. Use `@functools.cached_property` for instance-level caching instead.

### `@functools.cached_property`

Evaluates the property once on first access and caches the result on the instance. Avoids the GC issue of `@lru_cache` on methods:

```python
from functools import cached_property
from pathlib import Path

class Document:
    def __init__(self, path: Path) -> None:
        self._path = path

    @cached_property
    def content(self) -> str:
        """Read and cache file content. Evaluated once, then cached."""
        return self._path.read_text(encoding="utf-8")
```

Note: `cached_property` is incompatible with `@dataclass(frozen=True)` (frozen instances disallow attribute setting).

### `functools.partial`

Fix one or more arguments of a callable, returning a new callable:

```python
from functools import partial

def power(base: float, exponent: float) -> float:
    return base ** exponent

square = partial(power, exponent=2.0)
cube   = partial(power, exponent=3.0)

square(4.0)  # → 16.0
cube(3.0)    # → 27.0
```

## `itertools` — lazy iteration

**Golden rule: prefer generators and `itertools` over materialising lists until you need random access or `len()`.**

```python
from itertools import chain, islice, groupby, product, zip_longest, batched
```

| Function | Use |
|----------|-----|
| `chain(*iterables)` | Concatenate multiple iterables without copying |
| `islice(iterable, n)` | Take first `n` items from any iterable |
| `groupby(iterable, key)` | Group consecutive items by a key function |
| `product(*iterables)` | Cartesian product (nested loops as an iterator) |
| `zip_longest(*iterables, fillvalue=None)` | Zip unequal-length iterables |
| `batched(iterable, n)` | Chunk an iterable into batches of size `n` (Python 3.12+) |

```python
# chain — flatten without copying
all_tags = list(chain(post.tags for post in posts))  # ❌ wraps generator
all_tags = list(chain.from_iterable(post.tags for post in posts))  # ✅

# islice — take first 10 from a large generator
first_ten = list(islice(generate_records(), 10))

# batched — process in chunks (Python 3.12+)
for batch in batched(records, 100):
    db.insert_many(batch)

# groupby — requires pre-sorted input
from operator import attrgetter
sorted_items = sorted(items, key=attrgetter("category"))
for category, group in groupby(sorted_items, key=attrgetter("category")):
    process_group(category, list(group))
```

## `match`/`case` — structural pattern matching (Python 3.10+)

Replace complex `isinstance` chains with `match`:

```python
# ❌ isinstance chain
def handle(event: Event) -> str:
    if isinstance(event, ClickEvent):
        return f"click at {event.x},{event.y}"
    elif isinstance(event, KeyEvent) and event.key == "enter":
        return "enter pressed"
    elif isinstance(event, KeyEvent):
        return f"key: {event.key}"
    else:
        return "unknown"

# ✅ match/case
def handle(event: Event) -> str:
    match event:
        case ClickEvent(x=x, y=y):
            return f"click at {x},{y}"
        case KeyEvent(key="enter"):
            return "enter pressed"
        case KeyEvent(key=k):
            return f"key: {k}"
        case _:
            return "unknown"
```

**Rules:**
- Always include `case _:` — omitting it means unmatched values silently fall through.
- Use guard clauses (`case x if x > 0:`) for additional conditions.
- `match` is exhaustive only if you include `case _:` — mypy/pyright can verify exhaustiveness.

## `contextlib` — context manager utilities

### `@contextmanager`: the 90% solution

Write a context manager as a generator function instead of a full class:

```python
import logging
import time
from contextlib import contextmanager
from collections.abc import Generator

logger = logging.getLogger(__name__)

@contextmanager
def timer(label: str) -> Generator[None, None, None]:
    """Log elapsed time for the block."""
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        logger.debug("%s: %.3fs", label, elapsed)

with timer("data load"):
    data = load_large_dataset()
```

The code before `yield` is `__enter__`; the code after (in `finally`) is `__exit__`. Always put cleanup in `finally` so it runs even if the block raises.

### `ExitStack`: dynamic context manager composition

When you need to enter a variable number of context managers, use `ExitStack`:

```python
from contextlib import ExitStack
from pathlib import Path

def merge_files(paths: list[Path], output: Path) -> None:
    """Merge multiple files into one, opening only as many as needed."""
    with ExitStack() as stack:
        handles = [stack.enter_context(p.open(encoding="utf-8")) for p in paths]
        with output.open("w", encoding="utf-8") as out:
            for handle in handles:
                out.write(handle.read())
```

Without `ExitStack`, you would need deeply nested `with` blocks or try/finally chains.

For async context managers, use `asynccontextmanager` from `contextlib` — see `async.md`.
