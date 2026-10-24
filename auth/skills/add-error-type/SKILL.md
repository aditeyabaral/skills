---
name: add-error-type
description: Add a new failure mode to pesu-auth as a PESUAcademyError subclass with the right status code, message, optional headers, OpenAPI documentation and tests, so it renders the standard error body and is counted in errors_total. Use when a new condition needs its own status or message.
---

# Adding an error type

Error handling rules: `.agents/pesudev-skills/auth/docs/architecture.md` (Error handling) and
`.agents/pesudev-skills/auth/docs/api-contract.md` (status codes).

## 1. Choose the status

| Situation                                                       | Status                                                          |
| --------------------------------------------------------------- | --------------------------------------------------------------- |
| The caller sent something wrong that validation cannot express  | 400                                                             |
| Credentials or a token are wrong or missing                     | 401 (+ `WWW-Authenticate` for token auth)                       |
| Caller may not do this                                          | 403                                                             |
| PESU Academy's page cannot be parsed (it changed)               | 422 (reserved for this)                                         |
| PESU Academy unreachable, or answered with something unexpected | 502                                                             |
| Our bug                                                         | do not add a type; let it reach the 500 handler and fix the bug |

Changing the status of an existing error is breaking for callers who branch on it.

## 2. Define it

In `app/exceptions/<area>.py` (`authentication.py` for login/profile, `metrics.py` for metrics, or a
new module with a docstring for a new area):

```python
class UpstreamRateLimitedError(PESUAcademyError):
    """Raised when PESU Academy rejects requests for being too frequent."""

    def __init__(self, message: str = "PESU Academy is rate limiting requests. Please try again later.") -> None:
        """Initialize the UpstreamRateLimitedError with a custom message."""
        super().__init__(message, status_code=502)
```

- The default message is what callers see: safe to show, no internals, no personal data. Messages
  that include `user=...` are for logs; the handler returns `exc.message` to the caller, so check
  what you pass.
- Pass `headers={...}` only when HTTP requires it (for example `WWW-Authenticate` on a 401).
- No handler change is needed: `pesu_exception_handler` renders any subclass, logs 4xx at WARNING
  and 5xx with a traceback, and increments `errors_total{type=<class name>}`.

## 3. Raise it

Raise at the point the condition is detected. In `app/pesu.py`, if a metric reason also applies
(like `PROFILE_PARSE_ERRORS`), increment it just before raising, as the existing parse errors do.

## 4. Document it

- Add or extend the status in the route's `app/docs/<route>.py` `response_examples`, with
  `model: ResponseModel` and the exact default message as the example. If two errors share a status,
  use named `examples` (see the 502 in `app/docs/authenticate.py`).
- README: the route's "Responses" table, and the `errors_total{type}` list in the metrics "Failures"
  table.

## 5. Test it

- Unit: the condition raises the new type (`pytest.raises(NewError)`; assert on `exc_info.value`
  after the block).
- Route: through `TestClient` with the upstream patched to raise it; assert status, body
  `{status: False, message, timestamp}`, headers, and `errors_total{type="NewError"}` with a fresh
  collector.
- `uv run pytest tests/unit/test_openapi_docs.py -q`.
