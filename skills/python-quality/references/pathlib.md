# pathlib

Always use `pathlib.Path` for file system operations. Never use `os.path`.

## Import

```python
from pathlib import Path
```

## Common substitutions

| `os.path` (❌ avoid) | `pathlib.Path` (✅ use) |
|---------------------|------------------------|
| `os.path.join(a, b)` | `Path(a) / b` |
| `os.path.exists(p)` | `Path(p).exists()` |
| `os.path.isfile(p)` | `Path(p).is_file()` |
| `os.path.isdir(p)` | `Path(p).is_dir()` |
| `os.path.basename(p)` | `Path(p).name` |
| `os.path.dirname(p)` | `Path(p).parent` |
| `os.path.splitext(p)` | `Path(p).stem`, `Path(p).suffix` |
| `os.path.abspath(p)` | `Path(p).resolve()` |
| `os.makedirs(p, exist_ok=True)` | `Path(p).mkdir(parents=True, exist_ok=True)` |
| `open(p, "r")` | `Path(p).open("r")` or `Path(p).read_text()` |

## Reading and writing

```python
path = Path("data/input.txt")

# Read entire file
content = path.read_text(encoding="utf-8")
data = path.read_bytes()

# Write entire file
path.write_text("hello\n", encoding="utf-8")
path.write_bytes(b"\x00\x01")

# Append
with path.open("a", encoding="utf-8") as f:
    f.write("appended line\n")
```

## Building paths

```python
base = Path("/data/project")
output = base / "results" / "run_01.csv"  # ✅ / operator builds paths
```

## Common path operations

```python
p = Path("/data/project/src/main.py")

p.name       # "main.py"
p.stem       # "main"
p.suffix     # ".py"
p.parent     # Path("/data/project/src")
p.parts      # ("/", "data", "project", "src", "main.py")
p.resolve()  # absolute path with symlinks resolved

# Relative to current file — use in packages, not scripts
here = Path(__file__).parent
config = here / "defaults.toml"

# Iterate over directory
file_names = [child.name for child in Path("src").iterdir() if child.is_file()]

# Glob
python_files = list(Path("src").rglob("*.py"))

# Create directories
Path("output/results").mkdir(parents=True, exist_ok=True)

# Delete a file (does not raise if missing when using suppress)
from contextlib import suppress
with suppress(FileNotFoundError):
    Path("cache.tmp").unlink()
```

## Security: validate user-supplied paths

When a path comes from user input or untrusted data, always resolve and check it stays within the expected root:

```python
def safe_read(base: Path, user_path: str) -> str:
    """Read a file under base, rejecting traversal attempts."""
    resolved = (base / user_path).resolve()
    if not resolved.is_relative_to(base.resolve()):
        raise ValueError(f"path traversal attempt: {user_path!r}")
    return resolved.read_text(encoding="utf-8")
```

See `security.md` for more on path traversal.
