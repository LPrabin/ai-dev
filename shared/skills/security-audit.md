# Skill: Security Audit

Inspired by [Trail of Bits security skills](https://github.com/trailofbits/skills).

Static analysis and vulnerability detection as part of the dev workflow.

## When to Use
- Before any PR that touches auth, input handling, or external APIs
- When `ai-review` flags security concerns
- When adding new dependencies
- Periodically on the full codebase

## Automated Checks

Run these before committing security-sensitive code:

```bash
# Static security linter
uv run bandit -r src/ -c pyproject.toml

# Dependency vulnerability scan
uv run pip-audit

# Secret detection
git grep -nE '(password|secret|token|api_key)\s*=' -- '*.py' ':!*.example' ':!*test*'

# SQL injection patterns
git grep -nE "(f['\"].*SELECT|\.format\(.*SELECT|%s.*SELECT)" -- '*.py'

# Dangerous function usage
git grep -nE "(eval\(|exec\(|os\.system|shell=True|yaml\.load\(|pickle\.loads)" -- '*.py'
```

## Manual Review Checklist

### Input Boundaries
- [ ] All external input validated with Pydantic before use
- [ ] File paths: `Path(input).resolve()` then verify within allowed root
- [ ] File uploads: MIME type validated server-side

### Authentication & Secrets
- [ ] No hardcoded credentials in source
- [ ] Token comparison uses `hmac.compare_digest()`, not `==`
- [ ] Secrets from env vars / `pydantic-settings`, not config files
- [ ] No secrets in log output

### Data Handling
- [ ] SQL uses parameterized queries only
- [ ] No `yaml.load()` — use `yaml.safe_load()`
- [ ] No `pickle.loads()` on untrusted input
- [ ] Subprocess uses list args, never `shell=True` with variables

### Logging
- [ ] No passwords, tokens, PII, session IDs in logs
- [ ] Log user IDs and request IDs only

## Severity Levels
- **P0 (block release):** Injection, hardcoded secrets, auth bypass
- **P1 (fix this sprint):** Missing input validation, weak crypto
- **P2 (backlog):** Dependency CVE with mitigation, logging hygiene
