# Comments

When to write comments, what to say, and what not to say.

## The rule: explain *why*, not *what*

Code already shows *what* it does. A comment should explain *why* — the intent, the constraint, the non-obvious trade-off.

```python
# ❌ narrates the obvious
# Increment counter by 1
counter += 1

# ❌ restates the code
# Check if user is active and has a valid email
if user.active and user.email:

# ✅ explains a non-obvious reason
# We cap at 100 to stay within the API's rate limit (see ADR-0003).
MAX_REQUESTS_PER_MINUTE = 100

# ✅ explains a surprising implementation choice
# Using a list instead of a set here because insertion order matters
# for reproducible test output.
seen: list[str] = []
```

## When to write a comment

Write a comment when:

- The code is correct but **the reason is not obvious** from reading it.
- A constraint comes from **outside the code** (API limit, protocol quirk, performance measurement, ADR).
- You chose an **unusual approach** over the obvious one.
- There is a **known limitation or TODO** that future maintainers need to know.

## When NOT to write a comment

Do not comment:

- Code that is clear from the function name, variable names, and types.
- Obvious operations (`# open file`, `# return result`).
- Things that docstrings already cover — don't duplicate the docstring as an inline comment.

## TODO / FIXME conventions

```python
# TODO: replace with streaming API once rate limits are lifted (issue #42)
# FIXME: this is O(n²) — acceptable for n < 1000, must fix before scaling
```

**Rules:**
- `TODO` — work to do in the future, not urgent.
- `FIXME` — known bug or deficiency that needs addressing.
- Always include a reason or reference. Never leave bare `# TODO` or `# FIXME`.

## Block vs inline comments

```python
# Block comment: on its own line, same indentation as the code it describes.
# Describes what the NEXT block does at a high level.
records = fetch_all(source)

result = compute(x)  # Inline comment: short note on this specific line only.
```

**Rules:**
- Block comments: one blank line before the comment when it starts a new logical section.
- Inline comments: at least two spaces before `#`, used sparingly.
- Never put an inline comment on a `def`, `class`, or `with` statement — use a docstring instead.

## Style

- Full sentences with capital first letter and a period at the end (for multi-sentence comments; one-liners can omit the period).
- No emoji in source code comments.
- Keep comments up to date — a wrong comment is worse than no comment.
