# Data Structures

Decision guide for choosing between Python's typed data carriers.

## Decision guide

| Need | Use |
|------|-----|
| Simple data holder, no validation | `@dataclass` |
| Data with validation, serialisation, or schema | `pydantic.BaseModel` |
| Typed shape for a dict you don't own | `TypedDict` |
| Immutable record, tuple unpacking natural | `NamedTuple` |
| Structured data with unknown keys | `dict[str, X]` |

---

## `@dataclass` — simple data holders

Use when you own the data, there is no validation logic, and you want fast construction with zero dependencies.

```python
from dataclasses import dataclass, field

@dataclass
class BoundingBox:
    """An axis-aligned bounding box."""
    x_min: float
    y_min: float
    x_max: float
    y_max: float

@dataclass
class Config:
    """Runtime configuration."""
    host: str = "localhost"
    port: int = 8080
    tags: list[str] = field(default_factory=list)

# Immutable value object
@dataclass(frozen=True)
class Point:
    x: float
    y: float
```

**When to use:** internal DTOs, configuration objects, return values from pure functions.  
**When not to use:** when you need field-level validation, JSON serialisation, or a schema.

---

## `pydantic.BaseModel` — validated data

Use when data comes from an untrusted source (API, user input, config file) and needs validation, or when you need JSON serialisation / schema generation.

```python
from pydantic import BaseModel, Field

class UserRequest(BaseModel):
    username: str = Field(min_length=3, max_length=50)
    age: int = Field(ge=0, le=150)
    email: str

# Validated on construction — raises ValidationError on bad data
user = UserRequest(username="alice", age=30, email="a@example.com")
user.model_dump()        # → dict
user.model_dump_json()   # → JSON string
```

**When to use:** API request/response models, config loaded from files, any data crossing a trust boundary.  
**Requires:** pydantic as a runtime dependency or project-specific optional extra. See
`pydantic.md` for full patterns.

---

## `TypedDict` — typed dict shapes you don't own

Use when you receive a dict from an external source (API response, JSON payload) and want type safety without constructing your own class.

```python
from typing import TypedDict

class GithubUser(TypedDict):
    login: str
    id: int
    html_url: str

def get_username(user: GithubUser) -> str:
    return user["login"]  # mypy checks the key exists and is a str
```

`TypedDict` with `total=False` for optional keys:

```python
class Options(TypedDict, total=False):
    timeout: float
    retries: int
```

**When to use:** third-party API response shapes, configuration dicts passed around between functions.  
**When not to use:** when you own the data structure — use `@dataclass` instead.

---

## `NamedTuple` — immutable records

Use when you want tuple semantics (unpacking, positional access) with named fields and type safety.

```python
from typing import NamedTuple

class Coordinate(NamedTuple):
    latitude: float
    longitude: float
    altitude: float = 0.0

point = Coordinate(51.5, -0.1)
lat, lon, alt = point          # tuple unpacking works
point[0]                       # positional access works
```

**When to use:** small immutable records, return values from functions that return multiple related values, cases where tuple unpacking is a natural calling convention.  
**When not to use:** when you need methods, defaults on most fields, or mutability.

---

## Anti-pattern: plain `dict` for structured data

```python
# ❌ no type safety, no IDE support, easy to introduce typos
def process(config: dict) -> None:
    host = config["host"]      # KeyError at runtime if missing
    port = config["prot"]      # typo — no warning from mypy

# ✅ use a typed data carrier
def process(config: Config) -> None:
    host = config.host         # type-checked, autocompleted
    port = config.port
```

---

## Advanced dataclass options

### `@dataclass(slots=True)` — memory efficiency (Python 3.10+)

Generates `__slots__` automatically, eliminating the per-instance `__dict__`. Use it when many instances are expected or memory matters:

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Particle:
    x: float
    y: float
    z: float
    mass: float
```

With `slots=True`, each `Particle` uses ~40% less memory than a plain dataclass. See `class-design.md` for when to use `__slots__` manually.

### `@dataclass(frozen=True)` — immutability

Makes all fields read-only after construction. Instances become hashable and can be used as dict keys or set members:

```python
@dataclass(frozen=True)
class Coordinate:
    lat: float
    lon: float

coord = Coordinate(51.5, -0.1)
# coord.lat = 0.0  → FrozenInstanceError
{coord: "London"}  # hashable — works
```

Combine `frozen=True` and `slots=True` for a fully immutable, memory-efficient value object: `@dataclass(frozen=True, slots=True)`.

### `field(default_factory=...)` — mutable defaults

Never use a mutable literal as a default. Use `field(default_factory=...)`:

```python
from dataclasses import dataclass, field

# ❌ shared mutable default — all instances share the same list
@dataclass
class Bad:
    tags: list[str] = []  # ruff B006 will flag this

# ✅
@dataclass
class Good:
    tags: list[str] = field(default_factory=list)
    metadata: dict[str, str] = field(default_factory=dict)
```

### `__post_init__` — post-construction validation

Called automatically after the generated `__init__`. Use for validation or derived field computation:

```python
from dataclasses import dataclass

@dataclass
class PositiveRange:
    start: float
    end: float

    def __post_init__(self) -> None:
        if self.start >= self.end:
            raise ValueError(f"start ({self.start}) must be less than end ({self.end})")
```

For `frozen=True` dataclasses, use `object.__setattr__(self, "field", value)` inside `__post_init__` to set derived fields.

### `KW_ONLY` sentinel — keyword-only fields

Forces all fields after the sentinel to be keyword-only, preventing positional argument confusion in classes with many fields:

```python
from dataclasses import dataclass, KW_ONLY

@dataclass
class Request:
    url: str
    method: str
    _: KW_ONLY
    timeout: float = 30.0
    retries: int = 3
    verify_ssl: bool = True

# url and method can be positional; timeout, retries, verify_ssl must be keyword-only
req = Request("https://example.com", "GET", timeout=10.0)
```
