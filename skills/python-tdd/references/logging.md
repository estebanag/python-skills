# Logging

Stdlib `logging` conventions for this project.

## Module-level logger

Every module that emits log messages declares a module-level logger:

```python
import logging

logger = logging.getLogger(__name__)
```

`__name__` gives the logger a name matching the module's import path (e.g. `mypackage.processor`), which slots naturally into the logger hierarchy.

## Log levels

| Level | Use when |
|-------|----------|
| `DEBUG` | Detailed diagnostic information, useful only during development |
| `INFO` | Confirmation that things are working as expected |
| `WARNING` | Something unexpected happened, but the code is still working |
| `ERROR` | A serious problem — the code could not do what it was asked |
| `CRITICAL` | A very serious error; the program may be unable to continue |

```python
logger.debug("fetching %d records from %s", count, source)
logger.info("processing started for job %s", job_id)
logger.warning("retry %d/%d for request to %s", attempt, max_retries, url)
logger.error("failed to write output file: %s", path)
logger.critical("database connection lost — shutting down")
```

## Lazy evaluation: use `%s`, not f-strings

Log calls that use `%s` formatting are **only evaluated if the message is actually emitted**. f-strings are always evaluated, wasting CPU when the log level is filtered out.

```python
# ✅ lazy — string is built only if DEBUG is enabled
logger.debug("processing item: %s", item)

# ❌ eager — f-string is always evaluated, even if DEBUG is off
logger.debug(f"processing item: {item}")
```

## Exception logging

In `except` blocks, use `logger.exception()` to capture the full traceback:

```python
try:
    result = compute(data)
except ValueError as exc:
    logger.exception("compute failed for job %s", job_id)
    raise ProcessingError("computation failed") from exc
```

`logger.exception()` automatically includes the current exception's traceback. Equivalent to `logger.error(..., exc_info=True)`.

## Library code: never configure logging

This project is a library. **Library code must never call `logging.basicConfig()`, add handlers, or set levels.** That is the application's responsibility.

```python
# ❌ never do this in library code
logging.basicConfig(level=logging.DEBUG)
logging.getLogger().addHandler(logging.StreamHandler())

# ✅ just get a logger and use it
logger = logging.getLogger(__name__)
logger.info("library initialised")
```

If you are writing a CLI or application entry point (not a library module), you may configure logging there.

## Never use `print()` for operational output

`print()` bypasses the logging system. Use `logging` for anything that should be filterable or redirectable.

```python
# ❌ in library code
print(f"loaded {len(records)} records")

# ✅
logger.info("loaded %d records", len(records))
```

## Contextual information

For long operations or per-request context, use `logging.LoggerAdapter` or `extra`:

```python
logger.info("request processed", extra={"request_id": request_id, "user": user_id})
```
