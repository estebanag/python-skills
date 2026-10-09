# Testing

pytest conventions for this project.

## Running tests

```bash
uv run pytest                          # all tests
uv run pytest -v                       # verbose
uv run pytest tests/test_processor.py  # one file
uv run pytest -k "test_process"        # filter by name
uv run pytest --tb=short               # compact tracebacks
uv run pytest --pdb                    # drop into debugger on failure
```

## File naming

Tests live in `tests/` (flat — no subdirectories).

- One test file per source module: `src/mypackage/processor.py` → `tests/test_processor.py`
- If a module has many test scenarios, add a feature qualifier: `tests/test_processor_validation.py`

## Function naming

Format: `test_<verb>_<what>_<condition>`

```python
def test_process_items_returns_empty_on_empty_input(): ...
def test_process_items_raises_on_null_input(): ...
def test_validate_config_accepts_valid_dict(): ...
def test_validate_config_rejects_missing_required_key(): ...
def test_parse_date_handles_iso_format(): ...
```

The name should read as a specification sentence.

## Test structure: Arrange / Act / Assert

Separate the three phases with blank lines:

```python
def test_processor_returns_sorted_results() -> None:
    processor = Processor(max_results=3)

    result = processor.process(["c", "a", "b"])

    assert result == ["a", "b", "c"]
```

## Fixtures

Shared fixtures live in `tests/conftest.py`. Use fixtures for setup that multiple tests reuse.

```python
# tests/conftest.py
from __future__ import annotations

import pytest

from mypackage import Processor

@pytest.fixture
def processor() -> Processor:
    """Return a Processor configured for testing."""
    return Processor(max_results=10, strict=False)
```

```python
# tests/test_processor.py
def test_process_items_returns_sorted_results(processor: Processor) -> None:
    result = processor.process(["c", "a", "b"])

    assert result == ["a", "b", "c"]
```

## What makes a good test

**✅ Good tests:**
- Test behaviour through the **public API** — not internal implementation.
- Survive refactoring: renaming a private helper should not break any test.
- Are **fast and isolated** — no network, no file system, no shared mutable state.
- Are **readable as specifications**: the name and assertions tell you exactly what capability exists.

**❌ Bad tests:**
- Access private attributes (`processor._cache`) or call private methods.
- Share mutable state between test functions.
- Assert on internal calls rather than observable output.
- Have names like `test_foo` or `test_1`.

## Mocking

Use `unittest.mock.patch` (or `pytest-mock`'s `mocker`) **only for:**

- External I/O: network calls, file system, clocks, random numbers.
- Third-party services you cannot control in a test environment.

**Do NOT mock** internal collaborators of the code under test. If you're mocking a class defined in the same package, that's a sign the test is coupled to implementation.

```python
from unittest.mock import patch

def test_fetcher_retries_on_timeout() -> None:
    with patch("mypackage.fetcher.requests.get") as mock_get:
        mock_get.side_effect = TimeoutError
        fetcher = Fetcher(retries=2)

        with pytest.raises(FetchError):
            fetcher.fetch("https://example.com")

    assert mock_get.call_count == 2
```

## Parametrize

Use `@pytest.mark.parametrize` to avoid copy-paste tests:

```python
@pytest.mark.parametrize(
    ("input_val", "expected"),
    [
        ("hello", "HELLO"),
        ("", ""),
        ("123", "123"),
    ],
)
def test_shout_uppercases_input(input_val: str, expected: str) -> None:
    assert shout(input_val) == expected
```

## Async tests

This project uses `pytest-asyncio` with `asyncio_mode = "auto"` (configured in `pyproject.toml`). Write async test functions as `async def` — no decorator needed:

```python
# tests/test_fetcher.py
from __future__ import annotations

import pytest

from mypackage.fetcher import AsyncFetcher

async def test_fetch_returns_content() -> None:
    fetcher = AsyncFetcher(base_url="https://example.com")

    result = await fetcher.fetch("/api/data")

    assert result is not None
    assert len(result) > 0
```

### Async fixtures

Async fixtures work the same way — just make the fixture function `async def`:

```python
# tests/conftest.py
from __future__ import annotations

import pytest
from mypackage.client import AsyncClient

@pytest.fixture
async def client() -> AsyncClient:
    """Return an async client for testing."""
    async with AsyncClient() as c:
        yield c
```

### Mocking async functions

Use `AsyncMock` from `unittest.mock` when patching coroutines:

```python
from unittest.mock import AsyncMock, patch

async def test_processor_calls_fetch() -> None:
    with patch("mypackage.processor.fetch_data", new_callable=AsyncMock) as mock_fetch:
        mock_fetch.return_value = {"status": "ok"}
        processor = Processor()

        result = await processor.run()

    mock_fetch.assert_called_once()
    assert result["status"] == "ok"
```

## Test coverage

Coverage runs automatically on every `pytest` invocation (via `addopts` in `pyproject.toml`). The project enforces a minimum of **80% branch coverage**.

```bash
uv run pytest                          # runs tests + coverage report
uv run pytest --cov-report=html        # generate HTML report in htmlcov/
```

**Branch coverage** (`branch = true` in `[tool.coverage.run]`) detects untested conditional branches — a function with an untested `else` block will fail coverage even at 100% line coverage.

To mark genuinely untestable lines (e.g., defensive `raise NotImplementedError`):

```python
def abstract_method(self) -> str:
    raise NotImplementedError  # pragma: no cover
```

The following patterns are excluded from coverage automatically (configured in `pyproject.toml`):
- `pragma: no cover`
- `if TYPE_CHECKING:`
- `@overload`
- `raise NotImplementedError`
