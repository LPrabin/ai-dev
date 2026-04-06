# Skill: Systematic Debugging

Based on [superpowers/systematic-debugging](https://github.com/obra/superpowers).

No guessing. No random changes. Follow the phases.

## Phase 1 — Root Cause Investigation

1. **Read the error carefully.** Full traceback, top to bottom. Find YOUR code frame.
2. **Reproduce consistently.** If you can't reproduce, get more data before guessing.
3. **Check recent changes.** `git diff` and `git log --oneline -10`. What changed?
4. **Gather evidence at boundaries.** Log or print at component interfaces. Where does good data turn bad?
5. **Trace data flow.** Follow the value from origin to failure point.

## Phase 2 — Pattern Analysis

1. **Find a working example.** Same function, different input that works.
2. **Compare against reference.** Docs, tests, or known-good code.
3. **Identify the difference.** What's different between working and broken?

## Phase 3 — Hypothesis and Test

1. **Single hypothesis.** "The bug is X because Y." Evidence must support it.
2. **Minimal test.** Smallest possible change to confirm or deny.
3. **Verify before continuing.** If wrong, return to Phase 1 with new evidence. Don't stack guesses.

## Phase 4 — Implementation

1. **Write a failing test** that captures the exact bug.
2. **Single fix.** Smallest change that makes the test pass.
3. **Verify everything.** Run full test suite. No regressions.
4. **Document.** `Root cause: <what> — <why>`

## Red Flags — STOP if you catch yourself doing these

- Proposing a fix before reading the error
- Multiple "try this" changes without re-investigating
- Fixing symptoms instead of root cause
- 3+ failed fixes → stop and question your mental model
- Changing code you don't understand

## Tools

```bash
import pdb; pdb.set_trace()             # interactive debugger
uv run pytest -x --pdb                   # drop into pdb on failure
uv run mypy src/ --ignore-missing-imports # type errors
uv run python -m cProfile -s cumulative script.py | head -20  # profiling
PYTHONASYNCIODEBUG=1 uv run python script.py  # async debug
```

## Common Python Bugs

| Symptom | Likely cause |
|---|---|
| Mutation across tests | Mutable default arg `def f(lst=[])` |
| `None` where object expected | Missing `return` |
| Async not running | Missing `await` |
| Wrong values in loop | Closure over loop variable |
| Stale test data | Fixture scope too wide |
| Import errors | Circular imports |
