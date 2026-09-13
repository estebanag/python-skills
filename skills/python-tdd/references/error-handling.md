# Error Handling

Exception conventions for this project.

## Exception hierarchy

Every project defines a single root exception. All project-specific exceptions inherit from it. This lets callers catch all project errors with one `except` clause.

```python
# mypackage/exceptions.py

class MyPackageError(Exception):
    """Base exception for all mypackage errors."""

class ValidationError(MyPackageError):
    """Raised when input data fails validation."""

class NotFoundError(MyPackageError):
    """Raised when a requested resource does not exist."""

class ConfigurationError(MyPackageError):
    """Raised when the project is misconfigured."""
```

Re-export from `__init__.py` if callers need to catch them:

```python
# mypackage/__init__.py
from mypackage.exceptions import MyPackageError as MyPackageError
from mypackage.exceptions import NotFoundError as NotFoundError
from mypackage.exceptions import ValidationError as ValidationError

__all__ = [..., "MyPackageError", "ValidationError", "NotFoundError"]
```

## `raise X from Y`: exception chaining

When catching one exception and raising another, always chain them with `from`:

```python
try:
    raw = json.loads(data)
except json.JSONDecodeError as exc:
    raise ValidationError(f"invalid JSON in input: {data!r}") from exc
```

This preserves the original traceback and shows both exceptions when printed. **Never use bare `raise SomeError(...)` inside an `except` block without `from`** — it loses context.

To explicitly suppress the original exception (rare):

```python
raise ValidationError("invalid input") from None
```

## Never use bare `except`

```python
# ❌ catches SystemExit, KeyboardInterrupt, and every other exception
try:
    compute()
except:
    pass

# ❌ still too broad — catches MemoryError, RecursionError, etc.
try:
    compute()
except Exception:
    pass  # silently swallowed

# ✅ catch only what you can handle
try:
    compute()
except ValueError as exc:
    raise ValidationError("bad value") from exc
```

**Rule:** If you can't do something useful with an exception, let it propagate.

## Never silently swallow exceptions

An empty `except` block is almost always wrong. At minimum, log the exception:

```python
try:
    result = fetch_data()
except NetworkError as exc:
    logger.exception("failed to fetch data")
    raise  # re-raise after logging
```

`contextlib.suppress()` is acceptable only for genuinely ignorable errors:

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    cache_file.unlink()  # deleting a cache file — absence is fine
```

## Adding context to exceptions

Attach useful information to exceptions to aid debugging:

```python
class RecordNotFoundError(MyPackageError):
    def __init__(self, record_id: str) -> None:
        super().__init__(f"record {record_id!r} not found")
        self.record_id = record_id
```

## Where to define exceptions

- Shared exceptions: `mypackage/exceptions.py`
- Module-specific exceptions: in the module itself, prefixed with `_` if not public

## What NOT to use exceptions for

```python
# ❌ control flow — use dict.get() instead
try:
    value = my_dict[key]
except KeyError:
    value = default

# ✅
value = my_dict.get(key, default)

# ❌ validation in __init__ that raises a generic Exception
def __init__(self, x: int) -> None:
    if x < 0:
        raise Exception("x must be non-negative")  # too generic

# ✅ use a specific exception type
def __init__(self, x: int) -> None:
    if x < 0:
        raise ValueError(f"x must be non-negative, got {x}")
```
