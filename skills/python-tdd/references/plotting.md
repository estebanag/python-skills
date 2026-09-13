# Plotting

Matplotlib conventions for this project. Use these patterns only when the target project
already depends on Matplotlib, or when you have added Matplotlib to a project-specific optional
extra or runtime dependency.

```bash
uv add matplotlib
# or add it to a project-specific optional extra:
uv add --optional <extra> matplotlib
```

## Import convention

```python
import matplotlib.pyplot as plt
import matplotlib.figure
```

## Always use the object API

Use `fig, ax = plt.subplots()` — not the implicit `plt.plot()` state machine. The object API is explicit, composable, and safe in library code.

```python
# ✅ object API
fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(x, y, label="signal")
ax.set_xlabel("Time (s)")
ax.set_ylabel("Amplitude")
ax.set_title("Signal over time")
ax.legend()

# ❌ implicit state machine — unpredictable in library code
plt.plot(x, y)
plt.xlabel("Time (s)")
plt.show()
```

## Never call `plt.show()` in library code

`plt.show()` blocks execution and is for interactive sessions only. Library code must not call it — the caller decides what to do with the figure.

```python
# ❌ blocks execution; breaks non-interactive environments
plt.show()

# ✅ return the figure and let the caller save or display it
def plot_results(x: list[float], y: list[float]) -> matplotlib.figure.Figure:
    """Plot x vs y and return the figure."""
    fig, ax = plt.subplots(figsize=(8, 5))
    ax.plot(x, y)
    ax.set_xlabel("x")
    ax.set_ylabel("y")
    return fig
```

## Saving figures: always PNG, always descriptive names

```python
fig.savefig("loss_curve_epoch_100.png", dpi=150, bbox_inches="tight")
```

**Rules:**
- Format: **PNG** (not pdf, svg, or jpg unless there is a specific reason).
- Resolution: `dpi=150` for screen use; `dpi=300` for publication.
- `bbox_inches="tight"` prevents labels from being clipped.
- **Filename must describe the content.** It should be readable without opening the file.

```python
# ✅ descriptive filenames
"training_loss_vs_epoch.png"
"roc_curve_model_v2.png"
"feature_importance_top20.png"

# ❌ non-descriptive
"plot.png"
"figure1.png"
"output.png"
"test.png"
```

## Always close figures after saving

Matplotlib keeps figures in memory until explicitly closed. Always close after saving to avoid memory leaks in long-running scripts.

```python
fig, ax = plt.subplots()
ax.plot(data)
fig.savefig("output.png", dpi=150, bbox_inches="tight")
plt.close(fig)
```

## Always label axes and set a title

Every figure must have:
- `ax.set_xlabel(...)` with units if applicable
- `ax.set_ylabel(...)` with units if applicable
- `ax.set_title(...)` describing what the plot shows

```python
ax.set_xlabel("Epoch")
ax.set_ylabel("Cross-entropy loss")
ax.set_title("Training and validation loss over 100 epochs")
```

## Multiple subplots

```python
fig, axes = plt.subplots(nrows=2, ncols=2, figsize=(12, 8))
axes[0, 0].plot(x, y1, label="series A")
axes[0, 1].hist(values, bins=30)
axes[1, 0].scatter(x, y2, alpha=0.5)
axes[1, 1].bar(categories, counts)
fig.tight_layout()
fig.savefig("summary_grid.png", dpi=150, bbox_inches="tight")
plt.close(fig)
```
