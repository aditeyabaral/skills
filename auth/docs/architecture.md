# Architecture

How pesu-auth works, module by module, and the design decisions a change must not undo. Read the
section for the code you are about to touch before touching it.

## What the service does

pesu-auth is a stateless FastAPI service that checks PESU credentials by logging in to PESU Academy
(`https://www.pesuacademy.com`) on the caller's behalf, and optionally scrapes the student's profile
page. It stores nothing. PESU Academy is its **only** dependency: there is no database, cache
server or queue. Every counter lives in process memory.

Deployed on Render (free tier, Singapore) as a single uvicorn worker: production at
`https://pesuauth.onrender.com`, staging at `https://pesuauth-dev.onrender.com`.

## Module map

| Path                                    | Responsibility                                                                                        |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `app/app.py`                            | The FastAPI app: lifespan, OpenAPI override, middleware, exception handlers, all routes, `main()` CLI |
| `app/pesu.py`                           | `PESUAcademy`: the upstream client. CSRF prefetch/cache, login, profile scrape, client lifecycle      |
| `app/models/request.py`                 | `RequestModel`, the `/authenticate` body                                                              |
| `app/models/response.py`                | `ResponseModel`, the `/authenticate` success body (also the documented error body)                    |
| `app/models/profile.py`                 | `ProfileModel`, the `profile` object                                                                  |
| `app/models/metrics.py`                 | `MetricsModel` and friends: the `/metrics?fmt=json` view, built from a snapshot                       |
| `app/exceptions/base.py`                | `PESUAcademyError(message, status_code, headers)`, the base of every handled error                    |
| `app/exceptions/authentication.py`      | `AuthenticationError` 401, `CSRFTokenError` 502, `ProfileFetchError` 502, `ProfileParseError` 422     |
| `app/exceptions/metrics.py`             | `MetricsAuthorizationError` 401 with `WWW-Authenticate: Bearer`                                       |
| `app/metrics/collector.py`              | `MetricFamily` registry, `FAMILIES` render order, `MetricsCollector`, `MetricsSnapshot`               |
| `app/metrics/middleware.py`             | `record_request_metrics`: per-request counts, status, route, latency, in-flight                       |
| `app/metrics/prometheus.py`             | `MetricsFormat` enum and `render_prometheus` (text format 0.0.4)                                      |
| `app/metrics/auth.py`                   | `require_metrics_token`: optional bearer protection from `METRICS_TOKEN`                              |
| `app/docs/*.py`                         | `ApiDocs` request/response examples per route, fed to the route decorators                            |
| `scripts/run_tests.py`                  | The canonical test runner (coverage gate, live-test fallback)                                         |
| `scripts/benchmark/`                    | Load and CSRF-expiry benchmarks against a running instance                                            |
| `scripts/sync_agents.py`                | Generates `.github/agents/` from the roles in the submodule; `--check` is the `sync-agents` hook      |
| `.github/agents/*.agent.md`             | GitHub Copilot custom agents, one generated wrapper per role; never edited by hand                    |
| `.github/scripts/check_version_bump.py` | Used by the version-check workflow                                                                    |

Module-level singletons are created in `app/app.py`: `metrics = MetricsCollector()` and
`pesu_academy = PESUAcademy(metrics)`. Nothing else creates them; importing `app.metrics` has no
side effects. Tests swap them with `monkeypatch.setattr("app.app.metrics", ...)` and
`patch("app.app.pesu_academy....")`, which works because handlers look them up on the module at call
time. Keep it that way: never capture the singletons in a closure or default argument.

## Request lifecycle: `POST /authenticate`

1. **Middleware** (`metrics_middleware` → `record_request_metrics`) counts the request, bumps
   in-flight, starts a `perf_counter`.
1. **Validation**: FastAPI parses the body into `RequestModel` (`strict=True`, `extra="forbid"`,
   camelCase aliases). Failure raises `RequestValidationError` → `validation_exception_handler` →
   **400** (never FastAPI's default 422).
1. **Route** `authenticate()` records `AUTHENTICATION_REQUESTS{profile}` and calls
   `pesu_academy.authenticate(...)`.
1. **Upstream** (`PESUAcademy.authenticate`):
   1. `_get_client_with_csrf_token()` takes the prefetched client and token from the cache (hit) or
      fetches one inline (miss), then spawns a background prefetch for the next request.
   1. POSTs `_csrf`, `j_username`, `j_password` to `/Academy/j_spring_security_check`.
   1. If the response contains `div.login-form`, the login failed → `AuthenticationError` (401).
   1. Reads the post-login `meta[name='csrf-token']`, or raises `CSRFTokenError` (502).
   1. If `profile=True`, `get_profile_information()` GETs `/Academy/s/studentProfilePESUAdmin`
      and parses it (see `docs/upstream.md`), then filters to `fields` if they differ from
      `DEFAULT_FIELDS`.
   1. `finally`: closes the per-request client with `_close_client_quietly`.
1. **Route** records `AUTHENTICATION_RESULTS{result}`, validates the dict into `ResponseModel`,
   dumps it `by_alias=True, exclude_none=True`, and replaces `timestamp` with an ISO string.
   A `ValidationError` here becomes a **500** `PESUAcademyError`.
1. **Errors**: any `PESUAcademyError` → `pesu_exception_handler` → its `status_code` and headers,
   body `{status: false, message, timestamp}`. Anything else → `unhandled_exception_handler` → 500
   with a generic message.
1. **Middleware** records status, route template, latency and decrements in-flight in a `finally`.

Timestamps are always `datetime.now(IST)` with `IST = ZoneInfo("Asia/Kolkata")`.

## The CSRF prefetch design

Logging in needs an unauthenticated CSRF token from `GET /Academy/`. Fetching one per request adds a
full upstream round trip, so the service keeps **one** client and token ready:

- `lifespan` primes the cache at startup (`prefetch_client_with_csrf_token`; a failure here aborts
  startup, so the app cannot start without reaching PESU Academy) and starts
  `_csrf_token_refresh_loop`, which **sleeps first**, then refreshes every
  `CSRF_TOKEN_REFRESH_INTERVAL_SECONDS` (45 minutes). Sleeping first avoids a second fetch on
  startup.
- Each request **takes** the cached client and clears the cache under `_csrf_lock`, so no two
  requests share a client. A cold cache is filled by fetching **outside** the lock: holding it
  across a 10s upstream call would queue every concurrent request.
- After taking a client, the request spawns a background prefetch (`_spawn_prefetch_task`) for the
  next caller. Prefetch tasks are held in `self._prefetch_tasks` (strong references) and their
  failures are logged and counted in `_on_prefetch_task_done`, never raised.
- A prefetch that replaces a cached client closes the old one inside the lock.

## Client lifecycle and cancellation (the part most likely to regress)

Earlier releases leaked connection pools. The current code guarantees every `httpx2.AsyncClient`
is closed exactly once on every path, including cancellation during shutdown:

- `_fetch_new_client_with_csrf_token` closes the client if anything fails before it is returned
  (`except BaseException`, not `Exception`, so cancellation is covered).
- `_prefetch_client_with_csrf_token` closes the new client if it never reaches the cache.
- `authenticate` closes its per-request client in `finally`.
- `_close_client_quietly` runs the close as a shielded task held in `_CLOSE_TASKS`, so a second
  cancellation cannot abandon it half-way, and a failing close is logged and counted
  (`http_clients_total{event="close_failed"}`) instead of replacing the original error.
- `close_client()` (shutdown) cancels in-flight prefetches **before** closing the cached client, so
  a late prefetch cannot cache a client nobody closes.

`http_clients_total{created} - http_clients_total{closed}` should be **1** at rest (the cached
client). Any change to `app/pesu.py` must keep that true; see the `change-upstream-client` skill.

## Every upstream call is measured

Wrap every call to PESU Academy in `_upstream_call(self._metrics, "<operation>")` and append the
response to the yielded sink. It records `upstream_requests_total{operation,outcome}` (`success`,
`error`, `cancelled`), `upstream_latency_seconds{operation}` and, when a response exists,
`upstream_responses_total{operation,status}`. Operations today: `csrf_fetch`, `login`,
`profile_fetch`. A new upstream call needs a new operation name and the README and docs updates
listed in `docs/metrics.md`.

HTML parsing runs in a thread (`await asyncio.to_thread(HTMLParser, text)`) so a large page cannot
block the event loop. Keep that for any new parsing.

## Error handling

- Raise a `PESUAcademyError` subclass for every expected failure; never return error dicts.
- Status semantics: 4xx is the caller's problem (logged at WARNING, no traceback); 5xx is ours or
  upstream's (logged with `logging.exception`). The handler decides this from `status_code`.
- 502 means PESU Academy did not answer with what we expected; 422 is reserved for a profile page we
  could not parse (the page changed); 500 is a bug.
- `headers` on the exception is passed straight to the response (only the metrics 401 uses it).
- Each handler increments `errors_total{type=<exception class>}` exactly once. The middleware counts
  the status separately, so one failed request produces one error sample and one status sample.

## Middleware ordering facts

Starlette installs `ExceptionMiddleware` **below** user middleware and `ServerErrorMiddleware`
**above** it. So a handled error reaches `record_request_metrics` as an ordinary 4xx/5xx response,
while an unhandled exception arrives as an exception, and the middleware records a 500 itself
before re-raising. `scope["route"]` is only populated after `call_next`, so route labels are read
afterwards.

## OpenAPI override

`app.openapi` is replaced by `_openapi_without_phantom_validation_errors`, which deletes the
auto-generated 422 `HTTPValidationError` responses (this API returns 400 instead) but keeps the real
`/authenticate` 422 documented with `ResponseModel`. The schema is cached on `app.openapi_schema`.
Tests reset it with `app.openapi_schema = None`.

## Configuration

- CLI flags only for the server: `--host` (default `0.0.0.0`), `--port` (5000), `--debug`
  (DEBUG logging and uvicorn reload).
- The only environment variable the app reads is `METRICS_TOKEN`, **once at import**
  (`app/metrics/auth.py`). Unset or blank means `/metrics` is open.
- `.env` is read by the test suite and benchmark scripts only, never by the app.
- The Docker image (`Dockerfile`) installs with `uv sync --no-dev --frozen` from `uv.lock` and runs
  `python -m app.app`. It does not use `requirements.txt`.

## Scaling limits to keep in mind

`MetricsCollector` is deliberately unsynchronised: correct on one event loop, wrong under
`uvicorn --workers > 1` (per-process counters) or on a free-threaded build. The CSRF cache is also
per process. Anything that introduces multiple workers or threads must revisit both.
