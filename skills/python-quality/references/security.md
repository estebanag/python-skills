# Security

Security conventions for this project. These are non-negotiable defaults.

## Never use `shell=True`

`subprocess` with `shell=True` passes the command to the OS shell, enabling shell injection if any part of the command comes from untrusted input.

```python
# ❌ shell injection risk
subprocess.run(f"grep {user_input} /var/log/app.log", shell=True)

# ✅ pass arguments as a list — no shell involved
subprocess.run(["grep", user_input, "/var/log/app.log"])
```

**Rule:** Always pass commands as a `list[str]`. Never use `shell=True`.

## Never use `eval()` or `exec()`

Both execute arbitrary Python code. Never call them with input that comes from outside the program.

```python
# ❌ arbitrary code execution
result = eval(user_formula)

# ✅ use a safe expression parser or a whitelist of operations
```

## Path traversal prevention

Always resolve and validate paths to ensure they stay within the expected directory.

```python
from pathlib import Path

def read_user_file(base_dir: Path, filename: str) -> str:
    """Read a file from base_dir, rejecting traversal attempts."""
    resolved = (base_dir / filename).resolve()
    if not resolved.is_relative_to(base_dir.resolve()):
        raise ValueError(f"Path traversal detected: {filename!r}")
    return resolved.read_text(encoding="utf-8")
```

**Rule:** Always call `.resolve()` on user-supplied paths and check they are inside the expected root.

## Input validation

Validate and sanitise all data from external sources (user input, API responses, files, environment variables) before using it.

```python
# ✅ validate before use
def set_limit(value: str) -> None:
    try:
        limit = int(value)
    except ValueError:
        raise ValueError(f"limit must be an integer, got {value!r}")
    if limit < 1 or limit > 10_000:
        raise ValueError(f"limit must be between 1 and 10000, got {limit}")
    _apply_limit(limit)
```

Use pydantic validators for structured input (see `pydantic.md`).

## Cryptographic randomness

Use the `secrets` module for anything security-sensitive. Never use `random` for tokens, passwords, or IDs.

```python
import secrets

# ✅ cryptographically secure
token = secrets.token_hex(32)
password = secrets.token_urlsafe(16)

# ❌ not cryptographically secure
import random
token = str(random.getrandbits(256))
```

## Never log sensitive data

Passwords, tokens, API keys, and PII must never appear in log messages.

```python
# ❌ leaks credentials to log files
logger.info("connecting with password=%s", password)

# ✅ log only non-sensitive context
logger.info("connecting to %s as %s", host, username)
```

## Serialisation: prefer safe formats

Avoid `pickle` for untrusted data — it executes arbitrary Python during deserialisation.

```python
# ❌ arbitrary code execution on deserialise
data = pickle.loads(untrusted_bytes)

# ✅ use JSON, TOML, or other safe formats
import json
data = json.loads(untrusted_string)
```

Use `pickle` only for trusted, internal data where performance demands it.

## Dependency hygiene

- Pin dependencies with version bounds in `pyproject.toml` and commit `uv.lock`.
- Run `uv sync --upgrade` periodically to pick up security fixes.
- Check for known vulnerabilities: `uv run pip-audit` (if `pip-audit` is installed in the dev group).
