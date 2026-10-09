# NumPy Types

Type annotations for NumPy arrays. Use these patterns only when the target project already
depends on NumPy, or when you have added NumPy to a project-specific optional extra or runtime
dependency.

```bash
uv add numpy
# or add it to a project-specific optional extra:
uv add --optional <extra> numpy
```

NumPy ships its own type stubs — no separate install needed. mypy picks them up automatically.

## Import convention

```python
import numpy as np
import numpy.typing as npt
```

Always use the `npt` shorthand for `numpy.typing`.

## Annotating arrays

Use `npt.NDArray[dtype]` to annotate arrays with a specific dtype:

```python
import numpy as np
import numpy.typing as npt

def normalise(arr: npt.NDArray[np.float64]) -> npt.NDArray[np.float64]:
    """Normalise an array to the range [0, 1]."""
    min_val = arr.min()
    max_val = arr.max()
    return (arr - min_val) / (max_val - min_val)
```

## Common dtype annotations

| Type | Annotation |
|------|-----------|
| 64-bit float | `npt.NDArray[np.float64]` |
| 32-bit float | `npt.NDArray[np.float32]` |
| 64-bit integer | `npt.NDArray[np.int64]` |
| 32-bit integer | `npt.NDArray[np.int32]` |
| Boolean | `npt.NDArray[np.bool_]` |
| Complex 128-bit | `npt.NDArray[np.complex128]` |

## `npt.ArrayLike`: accepting flexible inputs

Use `npt.ArrayLike` for functions that accept lists, tuples, or arrays:

```python
def compute_mean(data: npt.ArrayLike) -> np.float64:
    """Compute the mean of any array-like input."""
    arr = np.asarray(data, dtype=np.float64)
    return arr.mean()
```

## `npt.DTypeLike`: dtype parameters

Use `npt.DTypeLike` when accepting a dtype as a parameter:

```python
def zeros(shape: tuple[int, ...], dtype: npt.DTypeLike = np.float64) -> npt.NDArray[np.float64]:
    """Create a zero-filled array with the given shape and dtype."""
    return np.zeros(shape, dtype=dtype)
```

## Shape documentation

mypy does not check array shapes. Document shapes in comments when they matter:

```python
def dot_product(
    a: npt.NDArray[np.float64],  # shape: (N,)
    b: npt.NDArray[np.float64],  # shape: (N,)
) -> np.float64:
    """Compute the dot product of two vectors."""
    return np.dot(a, b)

def matrix_multiply(
    a: npt.NDArray[np.float64],  # shape: (M, K)
    b: npt.NDArray[np.float64],  # shape: (K, N)
) -> npt.NDArray[np.float64]:  # shape: (M, N)
    """Multiply two matrices."""
    return a @ b
```

## Common patterns

```python
# Creating arrays
arr: npt.NDArray[np.float64] = np.array([1.0, 2.0, 3.0])
zeros: npt.NDArray[np.float64] = np.zeros((3, 4))
ones: npt.NDArray[np.int32] = np.ones((5,), dtype=np.int32)

# Type narrowing after asarray
def process(data: npt.ArrayLike) -> npt.NDArray[np.float64]:
    arr: npt.NDArray[np.float64] = np.asarray(data, dtype=np.float64)
    return arr * 2.0
```

## Vectorization

**The golden rule: if you are iterating over a numpy array with a Python `for` loop, stop and find the vectorized equivalent.**

Python loops over numpy arrays are 10–100× slower than equivalent vectorized operations. NumPy operations dispatch to optimised C code and can operate on entire arrays in a single call.

### Array operations over loops

```python
import numpy as np
import numpy.typing as npt

# ❌ Python loop — slow
def scale_loop(arr: npt.NDArray[np.float64], factor: float) -> npt.NDArray[np.float64]:
    result = np.empty_like(arr)
    for i in range(len(arr)):
        result[i] = arr[i] * factor
    return result

# ✅ vectorized — fast
def scale(arr: npt.NDArray[np.float64], factor: float) -> npt.NDArray[np.float64]:
    return arr * factor  # broadcasts scalar across entire array
```

### `np.where`: vectorized conditional

Replace element-wise `if/else` logic with `np.where`:

```python
# ❌ Python loop with conditional
def clip_negatives_loop(arr: npt.NDArray[np.float64]) -> npt.NDArray[np.float64]:
    result = np.empty_like(arr)
    for i in range(len(arr)):
        result[i] = arr[i] if arr[i] >= 0 else 0.0
    return result

# ✅ np.where
def clip_negatives(arr: npt.NDArray[np.float64]) -> npt.NDArray[np.float64]:
    return np.where(arr >= 0, arr, 0.0)
```

`np.where(condition, x, y)` returns elements from `x` where `condition` is `True`, else from `y`.

### `np.vectorize`: convenience, not performance

`np.vectorize` accepts a Python function and maps it over an array. It looks vectorized but **it is still a Python loop under the hood** — it provides no performance benefit over a manual loop. Use only when no native numpy operation exists and readability matters more than speed:

```python
# np.vectorize — convenience wrapper, same speed as a Python loop
lookup = np.vectorize(lambda x: category_map.get(x, "unknown"))
labels = lookup(codes)
```

For performance-critical code, find the native numpy operation or use `numba` / `cython`.
