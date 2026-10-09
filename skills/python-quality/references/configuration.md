# Configuration

How to manage configuration and environment variables in this project.

## Never hard-code environment-specific values

Hostnames, ports, API keys, database URLs, and feature flags must not appear as literals in source code.

```python
# ❌ hard-coded
DB_URL = "postgresql://prod-db.internal:5432/myapp"
API_KEY = "sk-prod-abc123"

# ✅ read from environment
import os
DB_URL = os.environ["DATABASE_URL"]
```

## `.env` files

Store local development values in a `.env` file at the project root. **Never commit `.env`** — it must be in `.gitignore`.

```
# .env  (gitignored)
DATABASE_URL=postgresql://localhost:5432/myapp_dev
API_KEY=sk-dev-xyz789
DEBUG=true
```

Always provide `.env.example` with all required keys but no real values:

```
# .env.example  (committed — shows what keys are needed)
DATABASE_URL=
API_KEY=
DEBUG=false
```

## `python-dotenv` for simple cases

Load `.env` at application startup:

```bash
uv add python-dotenv
```

```python
from dotenv import load_dotenv
import os

load_dotenv()  # loads .env into os.environ

database_url = os.environ["DATABASE_URL"]
debug = os.getenv("DEBUG", "false").lower() == "true"
```

**Rules:**
- Call `load_dotenv()` once, at the entry point — not in library modules.
- Use `os.environ["KEY"]` (raises `KeyError` if missing) for required values.
- Use `os.getenv("KEY", default)` only for genuinely optional values.

## `pydantic-settings` for validated configuration

For projects using pydantic, use `pydantic-settings` to get type coercion and validation.
Add it to the same runtime dependency set or project-specific optional extra as pydantic:

```bash
uv add pydantic-settings
# or, for a project-specific optional extra:
uv add --optional <extra> pydantic-settings
```

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
    )

    database_url: str
    api_key: str
    debug: bool = False
    max_workers: int = 4

# Instantiate once and pass around
settings = Settings()
```

Benefits over bare `os.environ`:
- Automatic type coercion (`"4"` → `int`)
- Validation via `Field()` constraints
- Clear documentation of all required env vars

## Accessing config in library code

Library code should receive configuration as constructor arguments or function parameters — not by reading `os.environ` directly. Reading env vars directly in library internals makes testing harder.

```python
# ❌ library reads env vars directly
class Processor:
    def __init__(self) -> None:
        self.api_key = os.environ["API_KEY"]  # hard to test

# ✅ configuration passed in
class Processor:
    def __init__(self, api_key: str) -> None:
        self.api_key = api_key
```

The application's entry point reads config from the environment and injects it:

```python
# main.py / cli.py
settings = Settings()
processor = Processor(api_key=settings.api_key)
```
