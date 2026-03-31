# Skill: Python patterns

## Project layout
```
project/
├── src/mypackage/
│   ├── models/       # Pydantic models, dataclasses
│   ├── services/     # Business logic — pure functions preferred
│   ├── api/          # FastAPI routers or CLI entry points
│   ├── db/           # Database layer, queries
│   └── utils/        # Shared helpers
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── pyproject.toml
├── .env.example
└── AGENTS.md
```

## pyproject.toml baseline
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mypackage"
version = "0.1.0"
requires-python = ">=3.12"

[project.optional-dependencies]
dev = ["pytest", "pytest-asyncio", "ruff", "black", "mypy", "bandit", "pip-audit"]

[tool.ruff]
line-length = 88
select = ["E","F","I","N","UP","B","S"]

[tool.black]
line-length = 88

[tool.mypy]
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

## Pydantic model pattern
```python
from pydantic import BaseModel, Field, field_validator
from decimal import Decimal

class PaymentRequest(BaseModel):
    amount: Decimal = Field(gt=0)
    currency: str = Field(min_length=3, max_length=3, pattern="^[A-Z]{3}$")
    idempotency_key: str

    @field_validator("currency")
    @classmethod
    def upper(cls, v: str) -> str:
        return v.upper()
```

## Service layer pattern
```python
from dataclasses import dataclass
from src.models.payment import PaymentRequest, PaymentResult
from src.db.payment_repo import PaymentRepository

@dataclass
class PaymentService:
    repo: PaymentRepository

    def process(self, request: PaymentRequest) -> PaymentResult:
        existing = self.repo.find_by_key(request.idempotency_key)
        if existing:
            return existing
        # ... business logic
```

## Safe subprocess
```python
import subprocess
from pathlib import Path

result = subprocess.run(
    ["git", "log", "--oneline", "-10"],
    capture_output=True, text=True, check=True,
    cwd=Path("/safe/path"),
)
# NEVER: subprocess.run(f"git {user_input}", shell=True)
```

## Error handling
```python
try:
    result = risky_operation()
except ValueError as e:
    logger.warning("Invalid input: %s", e)
    raise
except httpx.TimeoutException:
    raise ServiceUnavailableError("upstream timed out") from None
```

## Logging (safe)
```python
import logging
logger = logging.getLogger(__name__)
logger.info("Processing payment %s for user %s", payment_id, user_id)
# NEVER log tokens, passwords, or PII
```
