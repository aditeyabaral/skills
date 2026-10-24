---
name: change-upstream-client
description: Safely change how pesu-auth talks to PESU Academy in app/pesu.py - HTTP client creation and closing, timeouts, redirects, cancellation, error mapping and the measurement of each call - without reintroducing connection leaks, shared state between users or lost exceptions. Use for any change to the client lifecycle, concurrency or upstream request handling.
---

# Changing the upstream client

Earlier releases leaked connection pools, mixed sessions between users and let cleanup errors hide
real ones; most of the lines in this part of `app/pesu.py` exist because of one of those. Read "The
upstream client" in `.agents/pesudev-skills/auth/docs/architecture.md`, the flow in
`.agents/pesudev-skills/auth/docs/upstream.md`, and the comments in `app/pesu.py` before editing.

## Invariants you must preserve

1. **One client per login, used by nobody else.** `authenticate` creates the `httpx2.AsyncClient`;
   `_login` and `_fetch_profile` receive it. Never keep a client, a token or a response on `self` or
   at module level, and never prepare anything ahead of a request.
1. **Every client is closed exactly once, on every path**: success, each exception,
   `CancelledError`, and a cancellation arriving *during* the close. The close is
   `await _close_client_quietly(client, self._metrics)` in the `finally` of `authenticate`: a shielded
   task, strong-referenced in `_CLOSE_TASKS`, so it completes even when the request is cancelled.
1. **Cleanup failures never replace the original error.** `_aclose_client` logs and counts a failing
   close (`http_clients_total{event="close_failed"}`); a 401 must stay a 401.
1. **Every upstream request is inside `_upstream_call`** with an operation label, and the response
   is appended to the sink, so each call is counted and timed exactly once and its status recorded.
   Cancellation is `cancelled`, never `error`.
1. **Transport failures map to the step's error.** `httpx2.HTTPError` (timeouts, connection
   failures) is an `UpstreamError` during the login and a `ProfileFetchError` during the profile,
   raised `from e`.
1. **Timeout and redirects.** `UPSTREAM_TIMEOUT_SECONDS` (10) bounds how long a caller waits on
   PESU; redirects are not followed. Changing either changes user-visible behaviour and needs a reason
   in the PR.
1. **No upstream call outside a request**, so the app and `TestClient(app)` start without network
   access.
1. **Parse failures stay private.** Never log or chain a pydantic `ValidationError` from a PESU
   response; log `_validation_failure_summary(e)` and raise with `from None`.
1. **`http_clients_total` created minus closed is 0 at rest.**

## Testing each path

For every path you touch, write a test that fails if the invariant breaks. Existing examples in
`tests/unit/test_pesu.py` to copy:

- `test_each_login_gets_its_own_client_and_closes_it`, `test_the_client_is_closed_when_the_login_fails`,
  `test_the_client_is_closed_when_the_profile_fails`: close on success and on each error.
- `test_a_cancelled_profile_fetch_closes_the_client`, `test_a_cancellation_during_the_close_still_closes`:
  cancellation at each await, including during cleanup.
- `test_concurrent_logins_do_not_share_anything`: two logins in flight at once.
- `test_one_client_per_login_with_a_timeout_and_no_redirects`, `test_a_redirect_is_not_followed`,
  `test_a_timeout_is_an_upstream_error`, `test_the_login_request_on_the_wire`: through the `wire`
  fixture, so the real client builds and sends each request.
- `tests/unit/test_metrics_instrumentation.py`: the upstream and client metrics path by path.

Technique: the `upstream` fixture (patches `post`) or the `wire` fixture (a real client on a mock
transport), `make_response` for real `httpx2.Response`s, `asyncio.Event`s to hold a coroutine at a
known await, `task.cancel()` to cancel there, then assert the `http_clients_total` created and closed
counts on a fresh collector.

## Verifying for real

```bash
uv run python scripts/run_tests.py
uv run pytest -m secret_required -v                  # once, with credentials
uv run python -m app.app                             # then, in another shell:
curl -s "localhost:5000/metrics?fmt=json" | python -m json.tool | grep -A3 httpClients
```

After some logins and at rest, `created - closed` must be **0**. Stop the server with Ctrl+C and
check the log shows a clean shutdown with no "Task was destroyed" or "Unclosed client" warnings. For
load behaviour, run the `benchmark` skill with `--parallel` against the local server (small request
counts: it logs in to PESU for real) and check the client counts again.
