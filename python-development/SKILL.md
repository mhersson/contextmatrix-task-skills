---
name: python-development
description: Use when writing or modifying Python source files. Patterns for error handling, type hints, idioms, packaging, and testing.
---

You are a senior Python engineer. Match the surrounding code's style first; introduce new patterns only when the existing one is clearly worse for the task.

## Toolchain

The ContextMatrix Python worker image ships `uv`, `ruff` and `ty` (CPython 3.14,
no mypy, no preinstalled project dependencies). Use them unless the project pins
something else:

- `uv run <cmd>` / `uv add --dev <pkg>` for anything needing dependencies. Do not install into the managed interpreter under `/opt/python` - it is root-owned and the container runs as UID 1000.
- `ruff check` and `ruff format` for lint and formatting. Honour the project's `pyproject.toml` config; never add your own rule set.
- `ty check` after any signature change - `ty` is the installed type checker, not mypy.

## Errors

**Iron law:** raise specific exception types, catch the narrowest you can handle.

- Define custom exception classes when no stdlib type fits: `class CardNotFoundError(LookupError): ...`. Inherit from the closest stdlib base.
- Catch narrowly: `except json.JSONDecodeError:`, not `except Exception:`. Bare `except:` is almost always wrong.
- `from e` to chain context: `raise ConfigError("could not parse") from e`. Marks the original as the direct cause (`__cause__`) instead of leaving it as incidental `__context__`.
- Never `pass` an exception silently. If you genuinely don't care, log and explain in a comment.

## Type hints

**Iron law:** every public function has a complete type signature.

- `def parse(path: Path) -> Config:` — args and return type, always.
- `from __future__ import annotations` at the top of files with forward references.
- `X | None` (Py 3.10+) over `Optional[X]`. `Optional[X]` only when the project's already on it.
- `Sequence[X]` / `Mapping[K, V]` for parameters; `list[X]` / `dict[K, V]` for return types - accept broad, return specific (same idea as Go's accept interfaces, return concrete types).
- Don't use `Any` to silence the checker. Narrow with `cast` (sparingly), or fix the type.

## Idioms

- f-strings for formatting: `f"card {card.id} not found"`. No `%`-formatting in new code.
- Comprehensions for simple transformations: `[c.id for c in cards if c.state == "todo"]`. If it grows past one line, write a loop.
- `enumerate(items)` over `range(len(items))`.
- `with` for any resource (files, locks, connections). No bare `open()` without `with`.
- Pathlib over `os.path`: `Path("data") / "config.yaml"`.
- `@dataclass(frozen=True)` for plain data containers. `pydantic` only when validation/serialization is non-trivial AND the project already uses it.
- Lazy log formatting: `log.info("got %s", item)` — not f-strings inside log calls (eager formatting cost).

## Testing

- pytest, parametrized for tabular cases:

  ```python
  @pytest.mark.parametrize("input,expected", [
      ("42", 42),
      ("0", 0),
      ("-1", -1),
  ])
  def test_parse(input, expected):
      assert parse(input) == expected
  ```

- Fixtures (`@pytest.fixture`) for setup. Scope (`function` / `module` / `session`) is explicit.
- `tmp_path` and `monkeypatch` over global state; never mutate `os.environ` directly.
- Mock at the network/IO boundary (`responses`, `pytest-httpx`, `unittest.mock.patch`), not at the application layer.
- Plain `assert`. Pytest rewrites them for good failure messages.

## Imports & packaging

- Standard library, then third-party, then local — separated by blank lines.
- Absolute imports for clarity. Relative imports only for sibling modules in the same package.
- No code at import time (no I/O, no network). Module-level defines names only.

## Scope discipline

- Do what the task asks. Don't refactor surrounding modules, don't add helpers for hypothetical future use.
- Don't introduce a dependency for what the stdlib does. Three lines of `urllib.parse` beat a `requests` import.
- No half-finished implementations. Skip a sub-step entirely rather than stub it.
- Default to no comments. Type hints + clear names do most of the work.
- Don't add try/except for errors that can't actually happen.

## Quick red flags

| Red flag                                  | Why it's wrong                                              |
| ----------------------------------------- | ----------------------------------------------------------- |
| Bare `except:`                            | Also catches `KeyboardInterrupt`/`SystemExit`; use `except Exception:` if you must catch broadly |
| `except Exception:` around a whole block  | Hides bugs; catch the specific type you can handle          |
| `raise SomeError` without `from e`        | Reads as incidental "During handling..." context instead of the declared cause; use `from None` only to suppress it deliberately |
| Function with no return annotation        | Public API must be typed                                    |
| `Any` in a signature                      | Use a precise type or `cast` at the boundary                |
| Mutable default arg: `def f(x=[]):`       | Shared across calls; use `None` and assign inside           |
| `os.path.join` in new code                | Use `pathlib.Path`                                          |
| Module-level I/O (file read, network)     | Side effects on import                                      |
| `print()` for diagnostics in library code | Use `logging`                                               |
| `time.sleep()` in tests                   | Flaky; use `freezegun` / event waits                        |
