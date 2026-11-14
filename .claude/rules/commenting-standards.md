---
paths:
  - "**/*.py"
  - "**/*.pyi"
---

# Commenting Standards

These rules apply to any comment or docstring added, changed, or reviewed
in Python source under this repository. The goal is that every comment
earns its place: it tells a future reader (human or agent) something the
code alone doesn't. Examples below use Python; adapt the same principles
if the project later adds other languages.

## Block vs. inline comments

**Block comments** (standalone line, above the code) use full sentences
with capitalization and punctuation:

```python
# Bad - raises a KeyError if the setting was never configured.
timeout = settings["timeout"]

# Good - falls back to a sane default when the setting is unset.
timeout = settings.get("timeout", DEFAULT_TIMEOUT)
```

**Inline comments** (end of the code line) are lowercase, with no terminal
punctuation:

```python
batch_size = max(1, config.batch_size)  # never allow an empty batch
retries = settings.get("retries", 3)    # 3 matches the client's default
```

## Comment placement principles

**Data structure comments** (classes, dataclasses, `TypedDict`s):
- Belong at the **declaration** — a class docstring, or a comment directly
  above the field it documents.
- Explain the purpose, lifecycle, and invariants of the type.
- Document which code constructs/mutates each field and when the field
  becomes stale or invalid.
- Don't repeat this information in every method docstring that touches the
  type — document it once, at the declaration.

**Algorithmic comments:**
- Belong **inside function bodies**.
- Explain the logic, phases, and any non-obvious implementation detail.
- Focus on *why* the code does something, not a restatement of *what* it
  does.

**Function/method docstrings:**
- Focus on **inputs, outputs, and contract** — parameters, return value,
  exceptions raised, and side effects — not how the function is
  implemented internally.
- Document non-obvious preconditions or ordering requirements the caller
  must honor.
- Don't re-document data structure details already covered at the type's
  declaration.

**Module-level / design docstrings:**
- Must be understandable with **zero prior knowledge** of the module.
- Introduce any new term immediately, rather than assuming familiarity.
- Use a short example to ground an abstract concept.

## Comment types and examples

**Module-level design docstring:**

```python
"""Coordinates access to shared keys across concurrent transactions.

This module sequences transactions that read or write overlapping keys so
that locking and isolation guarantees hold without requiring callers to
manage locks directly.

The sequencer has two layers: a fast in-memory lock table for the common
case, and a wait queue for transactions that must block on a conflict.
"""
```

**Interface / protocol comments:**

```python
class AuthTransport(Protocol):
    """Interface the authenticator uses to talk to the client connection."""

    def send_auth_request(self, auth_type: int, data: bytes) -> None:
        """Ask the client for more authentication data.

        After calling this, the authenticator must call `read_auth_data`
        promptly — the connection's event loop blocks until the requested
        data arrives.
        """

    def read_auth_data(self) -> bytes:
        """Return the data previously requested via `send_auth_request`.

        Blocks until the data is available.
        """
```

**Function docstrings** focus on inputs, outputs, and contract:

```python
def append(self, clause: str, *params: object) -> None:
    """Append a SQL clause, rewriting its placeholders.

    Use the meta-placeholder ``$`` instead of a numbered placeholder;
    ``append`` renumbers all placeholders once every clause has been
    added.

    Example:
        query.append("SELECT * FROM foo WHERE a > $", low)
        query.append("AND a < $", high)
        # Produces: SELECT * FROM foo WHERE a > $1 AND a < $2

    Raises:
        ValueError: If the number of ``$`` placeholders doesn't match
            the number of params supplied.
    """
```

**Data structure / field comments:**

```python
@dataclass
class ConnectionPool:
    """Tracks reusable connections to a single backend host.

    Created once per host at startup and shared across requests. Callers
    must call `close()` during shutdown to release any open sockets.
    """

    host: str
    # Connections currently checked out by a caller; moved to `_idle`
    # when the caller releases them.
    _in_use: set[Connection] = field(default_factory=set)
    # Connections available for reuse, oldest first, so `acquire()` can
    # expire stale entries before handing one out.
    _idle: list[Connection] = field(default_factory=list)
```

**Phase comments in function bodies** separate processing steps and
explain non-obvious logic:

```python
def execute_with_retry(ctx: Context, op: Callable[[], Result]) -> Result:
    backoff = RetryPolicy.default()

    for _ in backoff.attempts(ctx):
        # The target may have been removed since the loop started.
        if not target_exists(ctx):
            raise TargetRemovedError()

        # Re-acquire the lease; it may have expired while we waited.
        if not has_lease(ctx):
            acquire_lease(ctx)

        try:
            return op()
        except RetryableError:
            continue

    raise TimeoutError("exhausted retries")
```

## Comment maintenance

- Add an explanation when you discover valuable knowledge that wasn't
  written down anywhere.
- Treat a factually incorrect comment or docstring as a bug — fix it
  immediately, not in a follow-up.
- Fix grammar or spelling that meaningfully impairs reading; skip purely
  cosmetic wording changes that don't improve understanding.
- When leaving grammar-only feedback in review, prefix it with "nit:" so
  it isn't mistaken for a correctness issue.

## No before/after diff narration

**Don't write comments or docstrings that describe what the code used to
do, or how this change differs from prior behavior.** That context belongs
in the commit message or PR description — not inline. A comment should
describe what the *current* code does, and only when that isn't
self-evident.

Code is read far more often by people (and agents) who only ever see its
current state than by people tracing its history. Diff narration ages
into confusing noise the moment another change lands.

```python
# Bad - narrates the diff; "previously" will mislead the next reader
# and goes stale as soon as another change lands.
# Previously this parsed timestamps with a hand-rolled regex. Now we
# use datetime.fromisoformat to support the full ISO 8601 format.
parsed = datetime.fromisoformat(raw_value)

# Good - explains why the current code does what it does.
# Use fromisoformat to accept the full range of ISO 8601 timestamps
# clients may send.
parsed = datetime.fromisoformat(raw_value)
```

Avoid these phrasings inline:

- "Previously, ..." / "We used to ..." / "Originally, ..."
- "No longer ..." / "This used to ..."
- "Replaced the old ... with ..."
- "Now we ..." / "Now does X (rather than Y)"
- "Without this, ..." / "Otherwise, would ..." (when describing the state
  *before* the change, rather than a true runtime invariant)
- "Instead of ..." (when it refers to a prior implementation rather than
  two choices that both exist today)

A counterfactual describing a *runtime* alternative is fine ("otherwise,
the caller would race with the writer" describes a real hazard, not a
diff). A counterfactual describing the *pre-change* state is diff
narration — drop it.

If historical context is genuinely load-bearing (e.g., a
backwards-compatibility shim that must stay in place), describe it in
terms of the *constraint*, not the *change*: "kept for compatibility with
clients still sending the v1 payload shape" is fine; "we used to send the
v1 payload shape" is not.

## How this is enforced

[`code-quality-reviewer`](../agents/code-quality-reviewer.md) checks new
and changed comments against this file during review, treats factually
incorrect or stale comments as **blocking**, and treats style violations
as **suggestion**/**nit** depending on severity.
