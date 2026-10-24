---
name: write-tests
description: Write pesu-auth tests that prove behaviour - choosing the suite, reusing the repo's fixtures for TestClient, PESU Academy mocked by post or on the wire, the fixture payloads, fresh metrics collectors and METRICS_TOKEN, testing errors, logs, metrics, cancellation and live flows - in the style the maintainers ask for. Use whenever adding or changing tests.
---

# Writing tests

Background and full recipes: `.agents/pesudev-skills/auth/docs/testing.md`.

## 1. Pick the suite

| Testing                                                                     | Suite                                              | Network         |
| --------------------------------------------------------------------------- | -------------------------------------------------- | --------------- |
| A function, model, handler, metric, or a route with `authenticate` patched  | `tests/unit/test_<module_or_concern>.py`           | none (required) |
| What a caller receives, through the whole app, with PESU mocked on the wire | `tests/unit/test_authenticate_flow.py`             | none (required) |
| `PESUAcademy` against the real PESU Academy                                 | `tests/functional/test_authenticate_functional.py` | live            |
| The whole app including lifespan, against the real PESU Academy             | `tests/integration/test_app_integration.py`        | live            |

Default to unit. Add a live test only when the behaviour depends on PESU's real responses, and mark
it `@pytest.mark.secret_required` if it needs the `TEST_*` credentials.

File and function names start with `test_` (pre-commit enforces it). Name tests for the behaviour:
`test_a_cancelled_profile_fetch_closes_the_client`, not `test_profile_2`.

## 2. Reuse the fixtures

Constants and shared fixtures live in `tests/unit/conftest.py` (and `tests/conftest.py` for the live
tests). Add new constants there, exposed through fixtures, never at the top of a test file.

```python
from unittest.mock import patch

import pytest
from fastapi.testclient import TestClient

from app.app import app
from app.exceptions.authentication import AuthenticationError
from app.metrics.collector import MetricsCollector


@pytest.fixture
def client(monkeypatch):
    monkeypatch.setattr("app.app.metrics", MetricsCollector())  # fresh counts per test
    with TestClient(app, raise_server_exceptions=False) as test_client:
        yield test_client


@patch("app.app.pesu_academy.authenticate")
def test_wrong_password_is_a_401_with_the_standard_body(mock_authenticate, client):
    mock_authenticate.side_effect = AuthenticationError(detail="Invalid credentials for user=u.")
    response = client.post("/authenticate", json={"username": "u", "password": "p"})
    assert response.status_code == 401
    body = response.json()
    assert body["status"] is False
    assert body["message"] == "Invalid username or password, or user does not exist."
    assert "timestamp" in body
```

For `PESUAcademy` directly, inject a collector and give the `upstream` fixture real responses in
call order (the login, then the profile):

```python
@pytest.fixture
def collector():
    return MetricsCollector(clock=lambda: 1000.0)


@pytest.fixture
def pesu(collector):
    return PESUAcademy(collector)


@pytest.mark.asyncio
async def test_x(pesu, collector, upstream, make_response, login_payload, profile_payload):
    profile_payload["STUDENT_INFO"]["SectionName"] = ""
    upstream.side_effect = [make_response(json=login_payload), make_response(json=profile_payload)]
    result = await pesu.authenticate("user", "pass", profile=True)
    assert result["profile"]["section"] is None
```

Use the `wire` fixture instead of `upstream` when the request itself matters (headers, multipart
fields, timeouts, redirects): it records every request a real client sends. Edit the deep-copied
`login_payload`/`profile_payload` per test; never put real personal data in a fixture.

## 3. What to assert

- **Status and full body** for route tests, not just the status.
- **Errors:** `with pytest.raises(X) as exc_info:` containing only the raising call; assert on
  `exc_info.value` (its `message` and `detail`) afterwards, never inside the block.
- **Nulls:** a field PESU has no value for is present and `None`, not absent.
- **Metrics:** exact series on a fresh collector, e.g.
  `collector.snapshot().value(UPSTREAM_REQUESTS.name, operation="login", outcome="success") == 1`.
- **Logs:** `caplog` for the message you expect, and that secrets are **absent**: the password, the
  access token and the `secrets` fixture values.
- **Cleanup:** `http_clients_total` created equals closed on each path for client code.
- **Docs:** new routes and examples are covered by `tests/unit/test_openapi_docs.py` and
  `tests/unit/test_openapi_contract.py` automatically; do not duplicate those checks.

## 4. Concurrency and cancellation

Use `asyncio.Event` to hold a coroutine at a known await, `asyncio.create_task` to run it,
`task.cancel()` to cancel there, and `pytest.raises(asyncio.CancelledError)` around the `await task`.
Assert state after (client closed, metric recorded). Copy the patterns in `tests/unit/test_pesu.py`
(`*cancel*`, `test_concurrent_logins_do_not_share_anything`).

## 5. Before you finish

- Run the new tests alone and with the suite: `uv run pytest <file> -v`, then `run-tests`.
- Break the code on purpose (comment out the fix) and confirm the test fails; restore it.
- `--cov-report=term-missing` shows no uncovered line or branch.

## Do not

Make unit tests hit the network; assert absolute counts on the shared `app.app.metrics` without
replacing it; test a mock instead of the code; add `# --- Section ---` markers; put real personal data
in fixtures; use `sleep` for synchronisation.
