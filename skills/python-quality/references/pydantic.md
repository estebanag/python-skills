# Pydantic

Pydantic v2 patterns for this project. Use these patterns only when the target project
already depends on pydantic, or when you have added pydantic to a project-specific optional
extra or runtime dependency.

```bash
uv add pydantic
# or add it to a project-specific optional extra:
uv add --optional <extra> pydantic
```

If using the mypy plugin, add it to `[tool.mypy]` in `pyproject.toml`:

```toml
[tool.mypy]
plugins = ["pydantic.mypy"]
```

## BaseModel

```python
from __future__ import annotations

from pydantic import BaseModel

class User(BaseModel):
    """A user account."""
    id: int
    name: str
    email: str
    active: bool = True
```

Models are validated on construction. Accessing `.model_fields` gives field metadata.

## Field constraints

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=200)
    price: float = Field(gt=0.0, description="Price in USD")
    quantity: int = Field(ge=0, le=10_000)
    tags: list[str] = Field(default_factory=list)
```

Common constraints: `min_length`, `max_length`, `gt`, `ge`, `lt`, `le`, `pattern`, `description`, `alias`.

## Field validators

```python
from pydantic import BaseModel, field_validator

class Config(BaseModel):
    host: str
    port: int

    @field_validator("port")
    @classmethod
    def port_must_be_valid(cls, v: int) -> int:
        """Validate that port is in the valid range."""
        if not (1 <= v <= 65535):
            raise ValueError(f"port must be 1–65535, got {v}")
        return v

    @field_validator("host")
    @classmethod
    def host_must_not_be_empty(cls, v: str) -> str:
        """Validate that host is non-empty after stripping."""
        v = v.strip()
        if not v:
            raise ValueError("host must not be empty")
        return v
```

## Model validators

Use `@model_validator` for cross-field validation:

```python
from pydantic import BaseModel, model_validator

class DateRange(BaseModel):
    start: str
    end: str

    @model_validator(mode="after")
    def end_must_be_after_start(self) -> DateRange:
        """Validate that end is after start."""
        if self.end <= self.start:
            raise ValueError("end must be after start")
        return self
```

## Private attributes

Use `PrivateAttr` for attributes that are not part of the model schema:

```python
from pydantic import BaseModel, PrivateAttr

class CachedProcessor(BaseModel):
    name: str

    _cache: dict[str, str] = PrivateAttr(default_factory=dict)

    def process(self, key: str) -> str:
        if key not in self._cache:
            self._cache[key] = key.upper()
        return self._cache[key]
```

## Model config

Use `model_config` with `ConfigDict` (not the v1 `class Config`):

```python
from pydantic import BaseModel, ConfigDict

class StrictModel(BaseModel):
    model_config = ConfigDict(
        frozen=True,  # immutable after creation
        str_strip_whitespace=True,
        validate_assignment=True,
    )
    name: str
```

## Serialisation / deserialisation

```python
# v2 API — use these
user.model_dump()  # → dict
user.model_dump(exclude={"password"})  # → dict without password
user.model_dump_json()  # → JSON string
User.model_validate({"id": 1, "name": "Alice", "email": "a@b.com"})  # from dict
User.model_validate_json('{"id": 1, "name": "Alice", "email": "a@b.com"}')

# ❌ v1 API — do not use
user.dict()
user.json()
User.parse_obj(data)
User.parse_raw(json_string)
```

## Settings management

For configuration loaded from environment variables, use `pydantic-settings` with the
`.env` file conventions in `configuration.md`.
