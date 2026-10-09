# Testing

Python testing conventions using pytest.

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

Follow the project's test layout. Name files for the public interface or behaviour under test. `tests/test_processor.py` is a useful default for a processor interface; use a qualifier such as `tests/test_processor_validation.py` when it makes a group of behaviours easier to find. A source module does not require its own test file.

## Function naming

A useful pattern is `test_<verb>_<what>_<condition>`:

```python
def test_process_items_returns_empty_on_empty_input(): ...
def test_process_items_raises_on_null_input(): ...
def test_validate_config_accepts_valid_dict(): ...
def test_validate_config_rejects_missing_required_key(): ...
def test_parse_date_handles_iso_format(): ...
```

Choose names that describe observable behaviour; use another pattern if it reads more clearly as a specification.

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
- Are **fast and isolated** — no uncontrolled external services or shared mutable state. Controlled temporary files and test databases are appropriate when the behaviour requires them.
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

Here the call count verifies the retry contract at an external I/O boundary. Do not assert on calls to internal collaborators.

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

Write async tests with `async def`. With `pytest-asyncio`, mark them with `@pytest.mark.asyncio`; the marker is optional if `asyncio_mode = "auto"` is enabled in the project's `pyproject.toml` (or other pytest configuration). Follow the project's async test plugin if it uses another convention:

```python
# tests/test_fetcher.py
from __future__ import annotations

import pytest

from mypackage.fetcher import AsyncFetcher

@pytest.mark.asyncio
async def test_fetch_returns_content() -> None:
    fetcher = AsyncFetcher(base_url="https://example.com")

    result = await fetcher.fetch("/api/data")

    assert result is not None
    assert len(result) > 0
```

### Async fixtures

Use `async def` fixtures when the async test plugin supports them. With `pytest-asyncio`, use its fixture decorator (a plain `@pytest.fixture` also works when automatic async fixture discovery is enabled):

```python
# tests/conftest.py
from __future__ import annotations

from collections.abc import AsyncIterator

import pytest_asyncio

from mypackage.client import AsyncClient

@pytest_asyncio.fixture
async def client() -> AsyncIterator[AsyncClient]:
    """Return an async client for testing."""
    async with AsyncClient() as c:
        yield c
```

### Mocking async functions

Use `AsyncMock` from `unittest.mock` when replacing an async call to an external service. Assert on the public result rather than the internal sequence of calls:

```python
from unittest.mock import AsyncMock, patch

import httpx

async def test_processor_returns_status_from_service() -> None:
    with patch("mypackage.processor.httpx.AsyncClient.get", new_callable=AsyncMock) as mock_get:
        mock_get.return_value = httpx.Response(
            200,
            json={"status": "ok"},
            request=httpx.Request("GET", "https://example.com/status"),
        )
        processor = Processor()

        result = await processor.run()

    assert result["status"] == "ok"
```

## Test coverage

Check the project's `pyproject.toml` (or other test configuration) for whether coverage runs with pytest, its required threshold, branch coverage, and exclusions. For example, pytest's `addopts` may enable coverage automatically, and `[tool.coverage.run]` may enable branch coverage. Branch coverage detects untested conditional paths even when line coverage is complete. Where coverage tooling is available, a detailed report can help identify missing paths:

```bash
uv run pytest --cov-report=html        # generate an HTML report when pytest-cov is configured
```

Mark genuinely untestable lines with `# pragma: no cover` only when the project's coverage configuration supports it, and explain why the line cannot be tested.
