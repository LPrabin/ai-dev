# Rules: Python

## Language and tooling
- Python 3.12+. Package manager: `uv`. Formatter: `black` (88). Linter: `ruff`. Types: `mypy --strict`.
- `uv add` to install. Never `pip install` directly in a project.
- `uv.lock` committed. `.python-version` committed.

## Types
- Type hints on all function signatures. `X | None` not `Optional[X]`. `list[str]` not `List[str]`.
- `Pydantic BaseModel` for all external data (API, config, CLI input).
- `dataclasses.dataclass` for internal value objects. Not plain dicts.
- No `Any` without `# type: ignore[reason]`.

## Structure
- `pathlib.Path` only. Never `os.path`.
- `Decimal` for money. Never `float`.
- Open files with explicit encoding: `open(path, encoding="utf-8")`.
- Context managers for all file, DB, network I/O.
- Imports: stdlib → third-party → local. Alphabetical within groups.

## Error handling
- Never bare `except:`. Catch specific exceptions only.
- Never swallow silently with `except Exception: pass`.
- Use `raise ... from e` when re-raising with context.

## Async
- Only for genuinely concurrent I/O-bound work.
- Don't add `async` speculatively.
- `asyncio_mode = "auto"` in pytest config.

## Forbidden
- `eval()`, `exec()` on non-literal input.
- `os.system()` or `subprocess.run(shell=True)` with variable content.
- SQL built with f-strings, `.format()`, or `%` interpolation.
- `yaml.load()` — use `yaml.safe_load()`.
- `pickle.loads()` on untrusted input.
- Hardcoded secrets, API keys, or passwords.
- `assert` for input validation (stripped with `-O`).
