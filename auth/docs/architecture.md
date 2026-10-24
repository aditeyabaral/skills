# Architecture

The design decisions in pesu-auth that a change must not undo, and the facts about its frameworks
that the code relies on. What the service is, its layout, how a request is handled and the table of
errors are in the repository's own `docs/architecture.md`, which `tests/unit/test_docs.py` keeps in
step with the code: read that first, then the section here for the code you are about to touch.

## What the service is

A stateless FastAPI service that checks PESU credentials by signing in to PESU Academy, through the
API its mobile app uses, on the caller's behalf, and optionally returns the student's profile. It
stores nothing. PESU Academy is its **only** dependency: no database, cache server or queue. Every
counter lives in process memory.

Deployed on Render (free tier, Singapore) as a single uvicorn worker: production at
`https://pesuauth.onrender.com`, staging at `https://pesuauth-dev.onrender.com`.

## Where things are

| Path                                    | Responsibility                                                                                                        |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `app/app.py`                            | The FastAPI app: lifespan, OpenAPI override, middleware, exception handlers, all routes, `main()` CLI                 |
| `app/pesu.py`                           | `PESUAcademy`: the login and profile calls, building the profile, the client lifecycle, `_upstream_call`              |
| `app/models/request.py`                 | `RequestModel`, the `/authenticate` body                                                                              |
| `app/models/response.py`                | `ResponseModel`, the `/authenticate` body (also the documented error body)                                            |
| `app/models/profile.py`                 | `ProfileField` (the field names a caller can request) and `ProfileModel`                                              |
| `app/models/upstream.py`                | The shapes read from PESU Academy: `LoginResponse`, `ProfileResponse`, `StudentInfo`, `StudentPhoto`, `ErrorEnvelope` |
| `app/models/metrics.py`                 | `MetricsModel` and friends: the `/metrics?fmt=json` view, built from a snapshot                                       |
| `app/exceptions/base.py`                | `PESUAcademyError(message, status_code, headers, detail)`, the base of every handled error                            |
| `app/exceptions/authentication.py`      | `AuthenticationError` 401, `UpstreamError` 502, `ProfileFetchError` 502, `ProfileParseError` 422                      |
| `app/exceptions/request.py`             | `RequestBodyParseError` 400, `RouteNotFoundError` 404, `MethodNotAllowedError` 405                                    |
| `app/exceptions/metrics.py`             | `MetricsAuthorizationError` 401 with `WWW-Authenticate: Bearer`                                                       |
| `app/metrics/`                          | `collector.py` (`MetricFamily`, `FAMILIES`, `MetricsCollector`), `middleware.py`, `prometheus.py`, `auth.py`          |
| `app/docs/*.py`                         | `ApiDocs` request and response examples per route, fed to the route decorators                                        |
| `app/templates/playground.html`         | The API Explorer at `/`: one self-contained file, no build step                                                       |
| `docs/`                                 | The repository's human documentation: architecture, development, testing, metrics, deployment                         |
| `scripts/run_tests.py`                  | The canonical test runner (coverage, live-test fallback)                                                              |
| `scripts/benchmark/`                    | Load benchmarks against a running instance, and their analysis                                                        |
| `scripts/sync_agents.py`                | Generates `.github/agents/` from the roles in the submodule; `--check` is the `sync-agents` hook                      |
| `.github/agents/*.agent.md`             | GitHub Copilot custom agents, one generated wrapper per role; never edited by hand                                    |
| `.github/scripts/check_version_bump.py` | Used by the *Version check* job                                                                                       |

## Module singletons and patch points

`app/app.py` creates `metrics = MetricsCollector()` and `pesu_academy = PESUAcademy(metrics)`.
Nothing else creates them; importing `app.metrics` has no side effects. Tests swap them with
`monkeypatch.setattr("app.app.metrics", ...)` and `patch("app.app.pesu_academy....")`, which works
because the middleware, handlers and routes look them up on the module at call time. Keep it that
way: never capture the singletons in a closure or default argument. The same goes for
`app.metrics.auth.METRICS_TOKEN`, which `require_metrics_token` reads at call time, and for
`app.pesu.httpx2.AsyncClient`, which the `wire` and `upstream` test fixtures replace.

## The upstream client

How the calls go and how the profile is read: `.agents/pesudev-skills/auth/docs/upstream.md`. What
must stay true of the code (the `change-upstream-client` skill has the full list):

- **One client per login.** `PESUAcademy.authenticate` creates an `httpx2.AsyncClient`, uses it for
  that login's calls only, and closes it in a `finally` with `_close_client_quietly`. No session
  state can pass between users, and nothing is cached between logins.
- **Closed exactly once on every path**, cancellation included. `_close_client_quietly` runs the
  close as a shielded task held in `_CLOSE_TASKS`, so a second cancellation during cleanup cannot
  abandon it, and `_aclose_client` logs and counts a failing close
  (`http_clients_total{event="close_failed"}`) instead of letting it replace the original error.
  `created - closed` in `http_clients_total` is **0** at rest.
- **Every call is measured.** Each request to PESU Academy runs inside
  `_upstream_call(self._metrics, "<operation>")`, with the response appended to the yielded sink. It
  records `upstream_requests_total{operation,outcome}` (`success`, `error`, `cancelled`),
  `upstream_latency_seconds{operation}` and, when a response exists,
  `upstream_responses_total{operation,status}`. The operations are `login` and `profile_fetch`; a new
  one needs the updates listed in `.agents/pesudev-skills/auth/docs/metrics.md`.
- **Responses are parsed into declared fields only.** `app/models/upstream.py` declares just what
  the service returns, so the rest of PESU's response (photos, addresses, parents' details, marks)
  never reaches a log, an exception or a repr. `LoginResponse.access_token` has `repr=False`.
- **No upstream call outside a request.** The lifespan only counts startup and shutdown, so the app
  starts without reaching PESU Academy, and so does `TestClient(app)`.

## Error handling

- Raise a `PESUAcademyError` subclass for every expected failure; never return error dicts or build
  error `JSONResponse`s in routes.
- `message` is what the caller sees: the exception's fixed default, part of the contract. `detail` is
  for the log: the username, what PESU answered. `pesu_exception_handler` logs `detail or message`
  and returns `message`.
- Severity follows the status: 4xx is the caller's problem (logged at WARNING, no traceback); 5xx is
  ours or upstream's (`logging.exception`).
- 502 means PESU Academy did not answer with what we expected; 422 is reserved for a profile response
  we could not parse (their API changed); 500 is a bug.
- `http_error_handler` turns the framework's own `HTTPException`s into ours: 404
  `RouteNotFoundError`, 405 `MethodNotAllowedError` (with an `Allow` header built from every route
  matching the path), 400 `RequestBodyParseError` (a body FastAPI cannot read at all). So every error
  has the standard body and is counted under its own class.
- `headers` on the exception go straight to the response (the metrics 401 and the 405 use it).
- Each handler increments `errors_total{type=<exception class>}` exactly once. The middleware counts
  the status separately, so one failed request produces one error sample and one status sample.

## Middleware ordering facts

Starlette installs `ExceptionMiddleware` **below** user middleware and `ServerErrorMiddleware`
**above** it. So a handled error reaches `record_request_metrics` as an ordinary 4xx/5xx response,
while an unhandled exception arrives as an exception: the middleware records a 500 itself before
re-raising, and `unhandled_exception_handler` adds the security headers to its own response because
`security_headers_middleware` never sees it. `scope["route"]` is only populated after `call_next`,
so route labels are read afterwards.

## OpenAPI override

`app.openapi` is replaced by `_openapi_without_phantom_validation_errors`, which:

- deletes the auto-generated 422 `HTTPValidationError` responses (this API returns 400 instead) but
  keeps the real `/authenticate` 422, documented with `ResponseModel`;
- puts each route's documented examples back as written (`_restore_documented_examples`), because
  FastAPI drops every null from the schema, which would show a field with no value as absent.

The schema is cached on `app.openapi_schema`. Tests reset it with `app.openapi_schema = None`.

## Configuration

- CLI flags: `--host` (default `0.0.0.0`), `--port` (default from `PORT`, else 5000), `--debug`
  (DEBUG logging and uvicorn reload).
- The app reads two environment variables: `PORT` (through the `--port` default) and
  `METRICS_TOKEN`, **once at import** (`app/metrics/auth.py`). Unset or blank means `/metrics` is
  open. `tests/unit/test_docs.py` fails if `docs/development.md` misses one.
- `.env` is read by the test suite and the benchmark scripts only, never by the app.
- The Docker image (`Dockerfile`) is `python:3.14-slim-trixie`, installs from the lock with
  `uv sync --locked --no-dev`, copies only the virtualenv and `app/` into the runtime stage, runs as
  UID 10001, and starts `python -m app.app`.

## Scaling limits to keep in mind

`MetricsCollector` is deliberately unsynchronised: correct on one event loop, wrong under
`uvicorn --workers > 1` (per-process counters) or on a free-threaded build. Anything that introduces
multiple workers or threads must revisit it.
