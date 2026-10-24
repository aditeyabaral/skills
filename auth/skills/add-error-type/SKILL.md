---
name: add-error-type
description: Add a new failure mode to pesu-auth as a PESUAcademyError subclass with the right status code, fixed message, log-only detail, optional headers, OpenAPI documentation, docs/ tables and tests, so it renders the standard error body and is counted in errors_total. Use when a new condition needs its own status or message.
---

# Adding an error type

Error handling rules: `.agents/pesudev-skills/auth/docs/architecture.md` (Error handling) and
`.agents/pesudev-skills/auth/docs/api-contract.md` (status codes).

## 1. Choose the status

| Situation                                                          | Status                                                          |
| ------------------------------------------------------------------ | --------------------------------------------------------------- |
| The caller sent something wrong that validation cannot express     | 400                                                             |
| Credentials or a token are wrong or missing                        | 401 (+ `WWW-Authenticate` for token auth)                       |
| Caller may not do this                                             | 403                                                             |
| PESU Academy's profile response cannot be parsed (its API changed) | 422 (reserved for this)                                         |
| PESU Academy unreachable, or answered with something unexpected    | 502                                                             |
| Our bug                                                            | do not add a type; let it reach the 500 handler and fix the bug |

Changing the status of an existing error is breaking for callers who branch on it.

## 2. Define it

In `app/exceptions/<area>.py` (`authentication.py` for login/profile, `metrics.py` for metrics, or a
new module with a docstring for a new area):

```python
class UpstreamRateLimitedError(PESUAcademyError):
    """Raised when PESU Academy rejects requests for being too frequent."""

    def __init__(
        self,
        message: str = "PESU Academy is rate limiting requests. Please try again later.",
        *,
        detail: str | None = None,
    ) -> None:
        """Initialize the UpstreamRateLimitedError with a custom message, and a detail for the log."""
        super().__init__(message, status_code=502, detail=detail)
```

- The default message is what callers see, and part of the contract: fixed text, safe to show, no
  internals, no personal data. Raise with the default and put the specifics in `detail=`
  (`detail=f"PESU Academy answered 429 for user={username}."`), which the handler logs and never
  returns.
- Pass `headers={...}` only when HTTP requires it (for example `WWW-Authenticate` on a 401).
- No handler change is needed: `pesu_exception_handler` renders any subclass, logs `detail or message` (4xx at WARNING, 5xx with a traceback), and increments `errors_total{type=<class name>}`.

## 3. Raise it

Raise at the point the condition is detected. In `app/pesu.py`, if a metric reason also applies
(like `PROFILE_PARSE_ERRORS`), increment it just before raising, as the `response_structure` parse
error does. When it replaces a pydantic `ValidationError` from PESU's response, raise `from None`.

## 4. Document it

- Add or extend the status in the route's `app/docs/<route>.py` `response_examples`, with
  `model: ResponseModel` and the exact default message as the example. If two errors share a status,
  use named `examples` (see the 502 in `app/docs/authenticate.py`).
- README: the route's "Responses" table.
- `docs/architecture.md`: a row in the error table (`tests/unit/test_docs.py` fails without it), and
  `docs/metrics.md`: the class in the `errors_total{type}` row.

## 5. Test it

- Unit: the condition raises the new type (`pytest.raises(NewError)`; assert on `exc_info.value`
  after the block).
- Route: through `TestClient` with the upstream patched to raise it; assert status, body
  `{status: False, message, timestamp}`, headers, that the `detail` is logged and not returned, and
  `errors_total{type="NewError"}` with a fresh collector.
- `uv run pytest tests/unit/test_openapi_docs.py tests/unit/test_openapi_contract.py tests/unit/test_docs.py -q`.
