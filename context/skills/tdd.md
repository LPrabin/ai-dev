# Skill: TDD

## The cycle
1. Red — write a failing test that describes desired behaviour.
2. Green — minimum code to pass.
3. Refactor — clean up, keep green.

Never skip to green without a red test first.

## Test anatomy (pytest)
```python
import pytest
from decimal import Decimal
from unittest.mock import MagicMock
from src.services.payment_service import PaymentService
from src.models.payment import PaymentRequest

@pytest.fixture
def mock_repo():
    return MagicMock()

@pytest.fixture
def service(mock_repo):
    return PaymentService(repo=mock_repo)

class TestProcessPayment:
    def test_returns_existing_on_duplicate_key(self, service, mock_repo):
        existing = {"id": "pay_123", "status": "completed"}
        mock_repo.find_by_key.return_value = existing
        request = PaymentRequest(
            amount=Decimal("10.00"), currency="USD", idempotency_key="key-abc"
        )
        result = service.process(request)
        assert result == existing
        mock_repo.create.assert_not_called()
```

## conftest.py pattern
```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

@pytest.fixture(scope="session")
def db_engine():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

@pytest.fixture
def db_session(db_engine):
    with Session(db_engine) as session:
        yield session
        session.rollback()
```

## What to test
- Always: happy path, error path, edge cases (None, empty, 0, max).
- Always: anything touching external systems — mock the boundary.
- Skip: framework code, generated code, trivial getters.

## Run commands
```bash
uv run pytest -xvs                              # fail fast, verbose
uv run pytest --tb=short -q                    # CI mode
uv run pytest -k "test_payment"                # filter
uv run pytest --cov=src --cov-report=term-missing
```
