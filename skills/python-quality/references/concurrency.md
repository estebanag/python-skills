# Concurrency

Threading and process-based concurrency patterns for this project.

## When to use what

| Scenario | Use |
|----------|-----|
| I/O-bound parallel work (network, disk) | `ThreadPoolExecutor` |
| CPU-bound parallel work (computation) | `ProcessPoolExecutor` |
| Shared mutable state between threads | `threading.Lock` / `threading.RLock` |
| Thread-safe producer/consumer queue | `queue.Queue` |
| True async I/O (many concurrent connections) | `asyncio` (out of scope here) |

## `threading.Lock`

Protect shared mutable state from concurrent modification:

```python
import threading

class Counter:
    """Thread-safe counter."""

    def __init__(self) -> None:
        self._value = 0
        self._lock = threading.Lock()

    def increment(self) -> None:
        """Increment the counter by 1."""
        with self._lock:
            self._value += 1

    @property
    def value(self) -> int:
        """Return the current counter value."""
        with self._lock:
            return self._value
```

**Rules:**
- Always acquire locks with `with` — never call `.acquire()` / `.release()` manually.
- Keep the locked section as short as possible.
- Use `threading.RLock` if the same thread needs to acquire the lock recursively.

## `queue.Queue`: thread-safe producer/consumer

```python
import queue
import threading

def producer(q: queue.Queue[str]) -> None:
    for item in generate_items():
        q.put(item)
    q.put(None)  # sentinel to signal completion

def consumer(q: queue.Queue[str]) -> None:
    while True:
        item = q.get()
        if item is None:
            break
        process(item)
        q.task_done()

q: queue.Queue[str] = queue.Queue(maxsize=100)
t1 = threading.Thread(target=producer, args=(q,))
t2 = threading.Thread(target=consumer, args=(q,))
t1.start(); t2.start()
t1.join(); t2.join()
```

`queue.Queue` is thread-safe — no external lock needed.

## `concurrent.futures.ThreadPoolExecutor`

For I/O-bound tasks run in parallel:

```python
import logging
from concurrent.futures import ThreadPoolExecutor, as_completed

logger = logging.getLogger(__name__)

def fetch(url: str) -> str:
    import urllib.request
    with urllib.request.urlopen(url) as resp:
        return resp.read().decode()

urls = ["https://example.com", "https://python.org"]

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {executor.submit(fetch, url): url for url in urls}
    for future in as_completed(futures):
        url = futures[future]
        try:
            result = future.result()
        except Exception as exc:
            logger.warning("%s failed: %s", url, exc)
        else:
            logger.info("%s: %d bytes", url, len(result))
```

## `concurrent.futures.ProcessPoolExecutor`

For CPU-bound tasks that need to bypass the GIL:

```python
from concurrent.futures import ProcessPoolExecutor

def expensive(n: int) -> int:
    return sum(range(n))

with ProcessPoolExecutor() as executor:
    results = list(executor.map(expensive, [10_000_000, 20_000_000]))
```

**Rules:**
- Functions and arguments must be picklable (no lambdas, no local functions).
- Use `ProcessPoolExecutor` only when profiling confirms the bottleneck is CPU-bound.

## Common pitfalls

```python
# ❌ non-atomic compound operation — race condition
if key not in self._cache:  # check
    self._cache[key] = compute(key)  # update — another thread may have set it between check and update

# ✅ use a lock
with self._lock:
    if key not in self._cache:
        self._cache[key] = compute(key)

# ❌ sharing a mutable default argument between threads
def process(items: list[str] = []) -> None: ...  # shared across all calls

# ✅ use None as default
def process(items: list[str] | None = None) -> None:
    if items is None:
        items = []

# ❌ forgetting to join threads — daemon threads die silently
t = threading.Thread(target=worker)
t.start()
# missing t.join() — main thread exits, work may be lost

# ✅
t.start()
t.join()
```

## The GIL

CPython's Global Interpreter Lock (GIL) means only one thread executes Python bytecode at a time. Threads are therefore useful for I/O-bound work (where threads sleep waiting for I/O), but **do not provide true parallelism for CPU-bound work**. Use `ProcessPoolExecutor` for CPU parallelism.
