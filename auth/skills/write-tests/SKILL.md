---
name: write-tests
description: Write pesu-auth tests that prove behaviour - choosing the suite, reusing the repo's fixtures for TestClient, mocked PESU Academy HTTP, fresh metrics collectors and METRICS_TOKEN, testing errors, logs, metrics, concurrency and live flows - and avoiding the suite's known anti-patterns. Use whenever adding or changing tests.
---

# Writing tests

Background and full recipes: `.agents/pesudev-skills/auth/docs/testing.md`.

## 1. Pick the suite

| Testing                                                                | Suite                                              | Network         |
| ---------------------------------------------------------------------- | -------------------------------------------------- | --------------- |
| A function, model, parser, metric, handler, route with upstream mocked | `tests/unit/test_<module_or_concern>.py`           | none (required) |
| `PESUAcademy` against the real site                                    | `tests/functional/test_authenticate_functional.py` | live            |
| The whole app including lifespan, against the real site                | `tests/integration/test_app_integration.py`        | live            |

Default to unit. Add a live test only when the behaviour depends on PESU's real responses, and mark
it `@pytest.mark.secret_required` if it needs the `TEST_*` credentials.

File and function names start with `test_` (pre-commit enforces it). Name tests for the behaviour:
`test_prefetch_closes_new_client_when_cancelled_before_caching`, not `test_prefetch_2`.

## 2. Reuse the fixtures

```python
from unittest.mock import AsyncMock, patch

import pytest
from fastapi.testclient import TestClient

from app.app import app
from app.metrics.collector import MetricsCollector


@pytest.fixture
def client(monkeypatch):
    monkeypatch.setattr("app.app.metrics", MetricsCollector())  # fresh counts per test
    with (
        patch("app.app.pesu_academy.prefetch_client_with_csrf_token", new_callable=AsyncMock),
        patch("app.app.pesu_academy.close_client", new_callable=AsyncMock),
    ):
        with TestClient(app, raise_server_exceptions=False) as test_client:
            yield test_client


@patch("app.app.pesu_academy.authenticate")
def test_wrong_password_is_a_401_with_the_standard_body(mock_authenticate, client):
    from app.exceptions.authentication import AuthenticationError

    mock_authenticate.side_effect = AuthenticationError()
    response = client.post("/authenticate", json={"username": "u", "password": "p"})
    assert response.status_code == 401
    body = response.json()
    assert body["status"] is False
    assert body["message"] == "Invalid username or password, or user does not exist."
    assert "timestamp" in body
```

For `PESUAcademy` directly, inject a collector and mock HTTP:

```python
@pytest.fixture
def collector():
    return MetricsCollector(clock=lambda: 1000.0)


@pytest.fixture
def pesu(collector):
    return PESUAcademy(collector)


@pytest.mark.asyncio
@patch("app.pesu.httpx2.AsyncClient.get")
async def test_x(mock_get, pesu, collector):
    response = AsyncMock()
    response.text = '<meta name="csrf-token" content="t">'
    response.status_code = 200
    mock_get.return_value = response
    ...
```

Stacked `@patch` decorators inject bottom-up. Mock upstream HTML with small strings containing the
real selectors (`.agents/pesudev-skills/auth/docs/upstream.md`) and fake values only.

## 3. What to assert

- **Status and full body** for route tests, not just the status.
- **Errors:** `with pytest.raises(X) as exc_info:` containing only the raising call; assert on
  `exc_info.value` afterwards. Never put assertions after the raise inside the block.
- **Metrics:** exact series on a fresh collector, e.g.
  `collector.snapshot().value(UPSTREAM_REQUESTS.name, operation="login", outcome="success") == 1`.
- **Logs:** `caplog` for the message you expect, and that secrets are **absent**:
  `assert "hunter2" not in caplog.text`.
- **Cleanup:** `aclose` awaited exactly once on each path for client code.
- **Docs:** new routes and examples are covered by `tests/unit/test_openapi_docs.py` automatically;
  do not duplicate those checks.

## 4. Concurrency and cancellation

Use `asyncio.Event` to hold a coroutine at a known await, `asyncio.create_task` to run it,
`task.cancel()` to cancel there, and `await asyncio.gather(task, return_exceptions=True)` to finish.
Assert state after (cache empty, client closed, metric recorded). Copy the patterns in
`tests/unit/test_pesu.py` (`*cancelled*`, `*strong_reference*`).

## 5. Before you finish

- Run the new tests alone and with the suite: `uv run pytest <file> -v`, then `run-tests`.
- Break the code on purpose (comment out the fix) and confirm the test fails; restore it.
- `--cov-report=term-missing` shows no new uncovered lines in `app/`.

## Do not

Make unit tests hit the network; assert absolute counts on the shared `app.app.metrics` without
replacing it; test a mock instead of the code (`tests/unit/test_readme.py` is the example to avoid);
put real personal data in fixtures; use `sleep` for synchronisation.
