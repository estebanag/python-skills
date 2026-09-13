# Async

Asyncio patterns for this project. Read this before writing any `async def` code.

## The cardinal rule

**Never block the event loop.**

Blocking the event loop prevents all other coroutines from running. Symptoms: the app appears to freeze, timeouts fire unexpectedly, or throughput collapses. Common culprits: `time.sleep`, synchronous file I/O, CPU-heavy computation, `requests.get`.

## Entry point: `asyncio.run`

`asyncio.run` is the only correct way to start an async program from synchronous code:

```python
import asyncio

async def main() -> None:
    await do_work()

if __name__ == "__main__":
    asyncio.run(main())
```

**Never use** `loop.run_until_complete(...)` — it is a low-level API that bypasses lifecycle management.

**Never nest** `asyncio.run` inside a running event loop — it raises `RuntimeError`. If you hit this, you are calling sync code that tries to start its own loop from inside an async context. Use `asyncio.to_thread` instead (see below).

## Concurrent coroutines

### `asyncio.gather` — fire and collect results

```python
import asyncio

async def fetch(url: str) -> str:
    await asyncio.sleep(0)  # placeholder for real I/O
    return f"response from {url}"

async def main() -> None:
    results = await asyncio.gather(
        fetch("https://example.com"),
        fetch("https://python.org"),
        fetch("https://pypi.org"),
    )
    # results is a list in the same order as the arguments
    assert results == [
        "response from https://example.com",
        "response from https://python.org",
        "response from https://pypi.org",
    ]
```

### `asyncio.TaskGroup` — structured concurrency (Python 3.11+, preferred)

`TaskGroup` is safer than `gather`: if any task raises, the group cancels the rest automatically.

```python
async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        task_a = tg.create_task(fetch("https://example.com"))
        task_b = tg.create_task(fetch("https://python.org"))
    # both tasks are done here — exceptions propagate as ExceptionGroup
    assert task_a.result() == "response from https://example.com"
    assert task_b.result() == "response from https://python.org"
```

**Rule:** prefer `TaskGroup` over `gather` for new code (Python 3.11+). Use `gather` when you need to tolerate individual task failures (`return_exceptions=True`).

## `async with` and `async for`

```python
# async context manager
async with aiofiles.open("data.txt", encoding="utf-8") as f:
    content = await f.read()

# async iterator
async for record in async_db_cursor:
    process(record)
```

Implement your own async context managers with `asynccontextmanager` from `contextlib` — see the `contextlib` section in `idioms.md`.

## The sync/async boundary

### Calling blocking code from async: `asyncio.to_thread`

When you must call a blocking synchronous function from an async context, offload it to a thread pool:

```python
import asyncio
from pathlib import Path

async def read_large_file(path: Path) -> str:
    """Read a file without blocking the event loop."""
    return await asyncio.to_thread(path.read_text, encoding="utf-8")
```

`asyncio.to_thread` runs the function in `ThreadPoolExecutor` and awaits the result. Use it for: synchronous file I/O, `requests` calls, CPU-light blocking operations.

For CPU-heavy work that cannot be threaded, use `ProcessPoolExecutor` (see `concurrency.md`).

### Calling async code from sync: `asyncio.run`

If you are in synchronous code and need a single async result:

```python
result = asyncio.run(some_coroutine())
```

This only works if no event loop is already running in the current thread. If one is already running (e.g., inside a Jupyter notebook or a test), you cannot call `asyncio.run` — you need to use `await` instead, meaning you must make the caller async too.

### `nest_asyncio` anti-pattern

`nest_asyncio` patches the event loop to allow nested `asyncio.run` calls. It is a workaround for environments like Jupyter. **Do not use it in library code** — it mutates global state and can cause subtle concurrency bugs in callers.

## Common pitfalls

```python
# ❌ blocking the event loop with time.sleep
async def wait() -> None:
    time.sleep(1)  # blocks the entire event loop

# ✅
async def wait() -> None:
    await asyncio.sleep(1)

# ❌ blocking the event loop with requests
async def fetch(url: str) -> str:
    return requests.get(url).text  # synchronous HTTP — blocks

# ✅ use httpx or aiohttp, or offload with to_thread
async def fetch(url: str) -> str:
    import httpx
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.text

# ❌ forgetting to await a coroutine
async def main() -> None:
    result = fetch_data()  # returns a coroutine object, never executes it
    print(result)          # prints <coroutine object ...>

# ✅
async def main() -> None:
    result = await fetch_data()
    assert isinstance(result, str)
```
