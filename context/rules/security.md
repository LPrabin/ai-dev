# Rules: Security

Violations here are blocking — fix before any commit.

## Injection (always block)
- No `eval()` / `exec()` on non-literal input.
- No `subprocess.run(shell=True)` with variable content. Use list args.
- No SQL built with string formatting — parameterized queries only.
- No `yaml.load()` — `yaml.safe_load()` only.
- No `pickle.loads()` on untrusted input.

## Secrets (always block)
- No hardcoded credentials, tokens, or keys in source.
- No secrets in log output.
- Use env vars + `python-dotenv` or `pydantic-settings`.
- Token comparison: `hmac.compare_digest()`, never `==`.

## Input validation
- All external input validated with Pydantic before use.
- User-supplied file paths: `Path(user_input).resolve()` then verify within allowed root.
- File uploads: validate MIME type server-side, not just extension.

## Logging hygiene
- Never log: passwords, tokens, full card numbers, SSNs, session IDs.
- Log user IDs and request IDs. Not user data.

## Dependency audit
- Run `uv run pip-audit` before shipping any release.
- No packages with known CVEs unless explicitly documented with a mitigation.
