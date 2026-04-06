# Skill: Test-Driven Development

Based on [superpowers/test-driven-development](https://github.com/obra/superpowers).

## Iron Law

No production code without a failing test first.
Code written before test → delete it and start over.

## The Cycle

### 1. RED — Write ONE failing test
- Smallest possible test for the next behaviour.
- Run it. **Watch it fail.** If it passes, you misunderstand the code — investigate.
- The failure message must clearly describe what's missing.

### 2. GREEN — Minimum code to pass
- Write the simplest implementation that makes the test green.
- No extra features. No "while I'm here" improvements.
- Run tests. **All must pass.** If others break, fix before continuing.

### 3. REFACTOR — Clean up, stay green
- Extract duplication, improve names, simplify.
- Do NOT add behaviour during refactor.
- Run tests after every change. Stay green.

## Mandatory Verification

Each phase MUST include running tests and confirming the expected state:
- RED: test fails with expected message
- GREEN: all tests pass
- REFACTOR: all tests pass, no behaviour change

Skip verification → restart the cycle from RED.

## Common Rationalizations (all wrong)

| Excuse | Reality |
|---|---|
| "Too simple to test" | Simple code still breaks. Test it. |
| "I'll write tests after" | You won't. And you'll write worse tests. |
| "The test would just duplicate the code" | Then you're testing implementation, not behaviour. |
| "I need to see the shape first" | Spike in a branch, throw it away, TDD the real thing. |
| "It's just a refactor" | Run existing tests. If none cover it, add one first. |

## Red Flags

- Writing multiple tests before any implementation
- Implementation that passes tests "by accident"
- Refactoring that changes test expectations
- Skipping the RED verification step
- Test file growing much faster than implementation

## Run Commands

```bash
uv run pytest -xvs                              # fail fast, verbose
uv run pytest --tb=short -q                      # CI mode
uv run pytest -k "test_payment"                  # filter
uv run pytest --cov=src --cov-report=term-missing # coverage
```
