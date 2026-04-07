# Rules: General

## Planning
- State approach before writing code. Never modify 3+ files without a plan review.
- Ambiguous task? Ask one clarifying question — not three.

## Code quality
- Functions do one thing. If you can't describe it in a sentence, split it.
- Max ~30 lines per function. Prefer under 20.
- No nested functions more than 2 levels deep.
- Delete dead code. Don't comment it out.
- No `TODO` without a linked issue.

## Naming
- Functions: verb phrases — `fetch_user`, `validate_token`.
- Classes: noun phrases — `UserRepository`, `PaymentService`.
- Booleans: `is_`, `has_`, `can_` prefix.
- Constants: `UPPER_SNAKE_CASE`.
- No abbreviations unless universal (`id`, `url`, `api`).

## Git
- Format: `type(scope): message` e.g. `feat(auth): add JWT refresh rotation`
- Types: `feat` `fix` `refactor` `test` `chore` `docs` `perf`
- Never commit to `main`. Branch → PR → review → merge.
- Stage with `git add -p`, not `git add .`.
- Don't commit commented-out code, `__pycache__`, or `.env`.

## Testing
- New behaviour → new test. Always.
- Test behaviour, not implementation.
- No `time.sleep()` in tests. Mock time.
- No flaky tests — fix or delete.

## Dependencies
- Don't add a package for something achievable in 10 lines of stdlib.
- Check the license before adding.
- Pin versions. Commit the lockfile.
