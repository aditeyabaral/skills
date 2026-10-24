# Testing

How to run the pesu-auth test suite, what to watch out for, and the recipes for tests that fit. The
layout, the coverage rules, the live tests and the benchmarks are described in the repository's own
`docs/testing.md`; this file adds what an agent needs on top.

## Layout

| Path                                               | What lives there                                                                                                                                                                                                                 | Network                      |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| `tests/unit/`                                      | One file per concern: `test_pesu.py`, `test_authenticate_flow.py` (the whole app with PESU mocked on the wire), `test_app_unit.py`, `test_metrics_*.py`, `test_openapi_docs.py`, `test_openapi_contract.py`, `test_docs.py`, ... | None: PESU Academy is mocked |
| `tests/unit/conftest.py`                           | The upstream fixtures (`upstream`, `wire`, `make_response`, `multipart_fields`), the fixture payloads (`login_payload`, `profile_payload`, `full_profile`, `secrets`) and every constant the unit tests use                      |                              |
| `tests/functional/test_authenticate_functional.py` | `PESUAcademy` against the real PESU Academy                                                                                                                                                                                      | **Live**                     |
| `tests/integration/test_app_integration.py`        | The full app through `TestClient`, with its real lifespan                                                                                                                                                                        | **Live**                     |
| `tests/conftest.py`                                | `load_dotenv()`, `PROFILE_VARIABLES` (one `TEST_*` variable per profile field), the live-profile checks, the autouse `_metrics_token_unset`, and the forced order unit → functional → integration                                |                              |

The tests marked `@pytest.mark.secret_required` log in with the `TEST_*` account from `.env`, and
the ones that fetch a profile compare every field with its variable. The other functional and
integration tests still call pesuacademy.com (with invalid credentials), so only `tests/unit/` runs
offline.

## Running

```bash
uv run python scripts/run_tests.py                   # the canonical run: coverage gate, live-test fallback
uv run pytest tests/unit -q                          # offline, under a minute
uv run pytest tests/unit/test_pesu.py::test_name -v  # one test
uv run pytest -m "not secret_required"               # everything that needs no credentials (needs internet)
uv run pytest -m secret_required -v                  # the live tests only
uv run pre-commit run --all-files                    # every hook, the tests included
```

`scripts/run_tests.py` runs `pytest --cov --cov-report=term-missing --disable-warnings -v -s`.
Coverage is configured once, in `[tool.coverage]` in `pyproject.toml`: it measures `app/`,
`scripts/` and `.github/scripts/`, counts branches, and fails under **100%**. If any of
`TEST_EMAIL`, `TEST_PRN`, `TEST_SRN`, `TEST_MOBILE` or `TEST_PASSWORD` is missing, the runner adds
`-m "not secret_required"` and warns **"Live PESU tests skipped"** (also as a GitHub Actions
annotation and in the step summary). A run with that warning is a **partial** run; report it as
such, never as a full pass.

## The live tests

- PESU Academy allows several sessions per account, so concurrent logins do not fail each other. The
  rules are about load and honest timings instead: run the live tests when a change needs them, once,
  never in a loop, never alongside another live run or a benchmark, and never with `-n`.
- Never print, log or paste the values of `TEST_*` variables. The live checks name mismatched fields,
  never their values; keep it that way.
- Fork pull requests get no secrets, so CI runs the reduced suite on every PR and the live tests only
  on pushes to `dev`. A local live run is the only place a change to `app/pesu.py`,
  `app/models/upstream.py` or the `/authenticate` path is verified against PESU before merge. Do it
  when credentials are available.
- A live failure on a field comparison after a PESU app release usually means their API changed (see
  `.agents/pesudev-skills/auth/docs/upstream.md`), not that the code did.
- A new profile field needs a `TEST_*` variable named after it, in `.env.example`, `PROFILE_VARIABLES`
  and the test job's `env` in `.github/workflows/ci_checks.yml`; a maintainer adds the secret.

## Fixtures and mocking recipes (copy these)

**The app through `TestClient`.** The lifespan makes no upstream calls, so the real one is safe to
enter. Patch the upstream result per test:

```python
@pytest.fixture
def client():
    with TestClient(app, raise_server_exceptions=False) as client:
        yield client


@patch("app.app.pesu_academy.authenticate")
def test_x(mock_authenticate, client):
    mock_authenticate.side_effect = AuthenticationError()
    ...
```

**`PESUAcademy` with PESU mocked.** Inject a fresh collector, and give the `upstream` fixture (which
patches `httpx2.AsyncClient.post`) real responses from `make_response`, in call order:

```python
@pytest.fixture
def collector():
    return MetricsCollector(clock=lambda: 1000.0)


@pytest.fixture
def pesu(collector):
    return PESUAcademy(collector)


@pytest.mark.asyncio
async def test_x(pesu, upstream, make_response, login_payload, profile_payload):
    upstream.side_effect = [make_response(json=login_payload), make_response(json=profile_payload)]
    result = await pesu.authenticate("user", "pass", profile=True)
    ...
```

Edit the deep-copied `login_payload` and `profile_payload` per test rather than writing new
payloads; they are shaped like the real responses, with invented values, and include personal
fields (`secrets`) the service must drop, so tests can prove it does.

**What goes over the wire.** The `wire` fixture replaces `httpx2.AsyncClient` with real clients on a
recording `MockTransport`, so headers, multipart encoding, timeouts and redirect handling run for
real. Route each URL to a callable returning a response or an exception, then inspect
`wire.requests`, `wire.client_options` and the decoded `multipart_fields(request)`. Use it for
anything about the request itself (`test_the_login_request_on_the_wire`,
`test_a_redirect_is_not_followed`), and through the whole app in `tests/unit/test_authenticate_flow.py`,
whose `collector` fixture also swaps `app.app.pesu_academy._metrics`.

**Fresh metrics per test.** The collector is a module singleton, so asserting absolute counts on it
is order-dependent (and the forced ordering hides that locally):
`monkeypatch.setattr("app.app.metrics", MetricsCollector())`.

**METRICS_TOKEN.** The autouse fixture sets `app.metrics.auth.METRICS_TOKEN` to `None`. A test that
needs protection sets it: `monkeypatch.setattr("app.metrics.auth.METRICS_TOKEN", metrics_token)`.

**OpenAPI schema.** Reset the cache around generation: `app.openapi_schema = None` (see
`tests/unit/test_openapi_docs.py`).

**Throwaway routes.** To test the unhandled-exception path, tests mount an `APIRouter` with
`include_in_schema=False` onto the real app at import time (`tests/unit/test_metrics_endpoints.py`).
Reuse that pattern; keep the route out of the schema so the OpenAPI and contract tests stay valid.

**Logs.** Use `caplog` (`with caplog.at_level("DEBUG"):`) to assert what was logged, and what was
**not**: no password, no access token, none of the `secrets` values.

**Async and cancellation.** Mark coroutine tests `@pytest.mark.asyncio` (strict mode). Hold a
coroutine at a known await with `asyncio.Event`s, cancel it with `task.cancel()`, and assert the
cleanup (`test_a_cancellation_during_the_close_still_closes`).

## Style the maintainers ask for

- Every constant a test file uses lives in a `conftest.py` and reaches tests through fixtures
  (parametrized fixtures where a `parametrize` list would need one), as `tests/unit/conftest.py` does.
- No `# --- Section ---` marker comments in test files.
- Put assertions about an exception after its `with pytest.raises(...) as exc_info:` block, never
  inside it, where they do not run.
- Tests are linted by ruff like the rest of the code, except that annotations and docstrings are not
  required (`ANN` and `D1` are ignored under `tests/`).
- File and function names start with `test_` (the `name-tests-test` hook); name tests for the
  behaviour: `test_a_redirect_is_not_followed`, not `test_login_3`.

## What a change needs

- A regression test that fails before a bug fix and passes after.
- Tests for every new line and branch: the gate is 100% line and branch coverage.
- Metric assertions when a code path records metrics (`tests/unit/test_metrics_instrumentation.py`
  asserts the upstream and client metrics path by path).
- The OpenAPI and contract tests pass unchanged for a new route; they fail if its docs are missing
  (see `.agents/pesudev-skills/auth/docs/api-contract.md`).
- `tests/unit/test_docs.py` passes: a new error class, CLI option or environment variable must be
  in the tables in `docs/`, and every path a `docs/` page names must exist.
- A live test (`secret_required`) only when behaviour depends on PESU's real responses, and never
  with assertions on personal values other than those supplied in `TEST_*` variables.

## CI

The *CI Checks* workflow (`.github/workflows/ci_checks.yml`) runs the hooks in its *Pre-commit
hooks* job with `SKIP=pytest`, and the tests in *Test suite & coverage*
(`uv sync --locked --all-groups`, then `scripts/run_tests.py`). The `TEST_*` secrets reach that job
on pushes to `dev`, never on fork PRs. `tests/unit/test_sync_agents.py` compares the committed
`.github/agents/` with the submodule, and skips that one test when the submodule is not checked out.
See `.agents/pesudev-skills/auth/docs/ci-cd.md`.
