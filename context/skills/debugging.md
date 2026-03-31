# Skill: Debugging

No random changes. Follow the protocol.

## Protocol
1. Reproduce reliably — `uv run pytest path/to/test.py -xvs`. Can't reproduce? Get more info.
2. Read the full traceback top to bottom. Find YOUR code frame, not library frames.
3. Narrow the boundary — last point state was correct vs first point it wasn't.
4. State a hypothesis — "the bug is X because Y." Evidence must support it.
5. Write a failing test that captures the bug before fixing.
6. Fix minimally — smallest change that passes the test.
7. Verify — `uv run pytest -q`. All green including the new regression test.
8. Write root cause: `Root cause: <what> — <why>`

## Tools
```bash
# Interactive debugger
import pdb; pdb.set_trace()
uv run pytest -x --pdb        # drop into pdb on failure

# Type errors
uv run mypy src/ --ignore-missing-imports

# Slow code
uv run python -m cProfile -s cumulative script.py | head -20

# Async debugging
PYTHONASYNCIODEBUG=1 uv run python script.py
```

## Common Python bugs
| Symptom | Likely cause |
|---|---|
| Mutation across tests | Mutable default arg `def f(lst=[])` |
| `None` where object expected | Missing `return` |
| Async not running | Missing `await` |
| Wrong values in loop | Closure over loop variable |
| Stale test data | Fixture scope too wide |
| Import errors | Circular imports |
