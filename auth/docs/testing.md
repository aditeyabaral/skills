# Testing

How the pesu-auth test suite is organised, how to run it, and how to write tests that fit.

## Layout

| Path                                               | What lives there                                                                                                                              | Network                                                                            |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `tests/unit/`                                      | One file per module or concern: `test_pesu.py`, `test_app_unit.py`, `test_request_model.py`, `test_metrics_*.py`, `test_openapi_docs.py`, ... | None: upstream is mocked                                                           |
| `tests/functional/test_authenticate_functional.py` | `PESUAcademy` against the real PESU Academy                                                                                                   | **Live**                                                                           |
| `tests/integration/test_app_integration.py`        | The full app through `TestClient`                                                                                                             | **Live** (its `client` fixture runs the real lifespan, which fetches a CSRF token) |
| `tests/conftest.py`                                | `load_dotenv()`, the autouse `_metrics_token_unset` fixture, and forced ordering unit → functional → integration                              |                                                                                    |

11 tests are marked `@pytest.mark.secret_required`: they log in with the `TEST_*` credentials from
`.env`. Everything else runs without credentials, **but the functional and integration files still
need internet access to pesuacademy.com** (invalid-credential and validation tests go through the
real lifespan or login). The unit tests run fully offline.

Today: 276 tests, 100% line coverage of `app/`, about 6 seconds without the live tests.

## Running

```bash
uv run python scripts/run_tests.py                  # the canonical run: coverage gate, live-test fallback
uv run pytest tests/unit -q                          # fast, offline
uv run pytest tests/unit/test_pesu.py::test_name -v  # one test
uv run pytest -m "not secret_required"               # everything that needs no credentials
uv run pre-commit run --all-files                    # everything CI runs, tests included
```

`scripts/run_tests.py` runs `pytest --cov=app --cov-report=term-missing --cov-fail-under=95 --disable-warnings -v -s`. If `TEST_EMAIL`, `TEST_PRN`, `TEST_PHONE` or `TEST_PASSWORD` is missing, it
adds `-m "not secret_required"` and prints **"Live PESU tests skipped"**. A run with that line is a
**partial** run; report it as such, never as a full pass.

## The live tests and the single session

The PESU test account allows **one active session**. Two concurrent logins make one of them fail
with a confusing 401. Therefore:

- Run live tests **one run at a time**. Never run two test processes in parallel, never run pytest
  with `-n`/xdist, and never start a live run while another agent or terminal is running one.
- CI runs the live tests on pushes to `dev` (where secrets exist). Do not start a local live run
  right after something merged to `dev`.
- Never print, log or paste the values of `TEST_*` variables.
- Live runs send real traffic to PESU Academy. Do not loop them.

Fork pull requests get no secrets, so CI runs the reduced suite on every PR. That makes a local live
run the only place a change to login or scraping is verified before merge. Do it for any change to
`app/pesu.py` or to the `/authenticate` path when credentials are available.

## Fixtures and mocking recipes (copy these)

**App through `TestClient` without touching the network** (from `tests/unit/test_app_unit.py`):

```python
@pytest.fixture
def client():
    with (
        patch("app.app.pesu_academy.prefetch_client_with_csrf_token", new_callable=AsyncMock),
        patch("app.app.pesu_academy.close_client", new_callable=AsyncMock),
    ):
        with TestClient(app, raise_server_exceptions=False) as client:
            yield client
```

Then patch the upstream result per test: `@patch("app.app.pesu_academy.authenticate")` with a
`return_value` dict or a `side_effect` exception.

**Fresh metrics per test.** The collector is a module singleton, so asserting absolute counts
without a fresh one is order-dependent (and the forced ordering hides it locally):

```python
monkeypatch.setattr("app.app.metrics", MetricsCollector())
```

For `PESUAcademy` directly, inject one: `PESUAcademy(MetricsCollector(clock=lambda: 1000.0))`
(see `tests/unit/test_metrics_instrumentation.py`).

**Upstream HTTP.** Patch the client methods and give responses `text` and `status_code`:

```python
@patch("app.pesu.httpx2.AsyncClient.get")
@patch("app.pesu.httpx2.AsyncClient.post")
@pytest.mark.asyncio
async def test_x(mock_post, mock_get, pesu): ...
```

Mock decorators apply bottom-up, so the first argument is the **last** decorator. Build profile
pages as small HTML strings with the real selectors (`div.elem-info-wrapper`, `div.form-group`,
`label.lbl-title-light` + `label`, `#updateMail`, `#updateContact`); see `test_pesu.py`.

**METRICS_TOKEN.** The autouse fixture sets `app.metrics.auth.METRICS_TOKEN` to `None`. A test that
needs protection sets it: `monkeypatch.setattr("app.metrics.auth.METRICS_TOKEN", "some-token")`.

**OpenAPI schema.** Reset the cache around generation: `app.openapi_schema = None` (see
`tests/unit/test_openapi_docs.py`).

**Throwaway routes.** To test the unhandled-exception path, tests mount an `APIRouter` with
`include_in_schema=False` onto the real app at import time. Reuse that pattern; keep the route out
of the schema so the OpenAPI tests stay valid.

**Logs.** Use `caplog` (`with caplog.at_level("DEBUG"):`) to assert what was logged, and to assert
what was **not** logged (for example that a password never appears).

**Async.** Mark coroutine tests `@pytest.mark.asyncio` (pytest-asyncio is in strict mode by default).

## What a change needs

- A regression test that fails before a bug fix and passes after.
- Unit tests for every new branch. Coverage is 100% today; keep it there even though the gate is 95%.
- Metric assertions when a code path records metrics (`tests/unit/test_metrics_instrumentation.py`
  asserts upstream and cache metrics path by path).
- OpenAPI tests pass unchanged for a new route; they will fail if docs are missing (see
  `.agents/pesudev-skills/auth/docs/api-contract.md`).
- A live test (`secret_required`) only when behaviour depends on the real PESU page, and never with
  assertions on personal values other than those supplied in `TEST_*` variables.

## Anti-patterns found in the suite (do not copy)

- Asserting after the raising call inside `with pytest.raises(...)`: those lines never run. Put
  assertions on the exception (`exc_info.value`) after the block.
- `tests/unit/test_readme.py` patches `readme` and then calls the mock, so it tests nothing. Test
  real routes through `TestClient`.
- Asserting absolute metric counts on the shared singleton.

## CI

`pre-commit.yaml` runs `uv run pre-commit run --all-files` on every push and PR, which includes the
`pytest` hook (`python scripts/run_tests.py`, verbose so the skip warning is visible) and the
`sync-agents` hook (`python scripts/sync_agents.py --check`). The workflow checks out the
`.agents/pesudev-skills` submodule for the latter; `tests/unit/test_sync_agents.py` also compares
the committed `.github/agents/` with the submodule, and skips that one test when the submodule is
not checked out. Secrets are present on pushes to the base repository, not on fork PRs. See
`.agents/pesudev-skills/auth/docs/ci-cd.md`.
