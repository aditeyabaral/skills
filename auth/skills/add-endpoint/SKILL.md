---
name: add-endpoint
description: Add a new route to pesu-auth the way the repo requires - route in app/app.py with tags and docstring, pydantic models, an ApiDocs module with examples for every response, errors via PESUAcademyError, metrics, README section and tests that satisfy the OpenAPI documentation and contract tests. Use for any new HTTP endpoint.
---

# Adding an endpoint

Read `.agents/pesudev-skills/auth/docs/api-contract.md` first. Decide: method, path (lowercase, no
trailing slash), tag (`Authentication`, `Documentation` or `Monitoring`, or add a new tag to
`openapi_tags` in `app/app.py`), auth (open, or protected like `/metrics`), and response shape.

## 1. Models (if the endpoint takes a body or returns a new shape)

In `app/models/<name>.py`, export from `app/models/__init__.py`:

```python
class FooRequestModel(BaseModel):
    """Model representing ..."""

    model_config = ConfigDict(strict=True, alias_generator=to_camel, extra="forbid")

    some_field: str = Field(..., title="...", description="...", json_schema_extra={"example": "..."})
```

Responses use `populate_by_name=True` instead of `extra="forbid"`, and are dumped with
`model_dump(by_alias=True)` (`/authenticate` uses `exclude_unset=True` so that a value it has not got
is `null` rather than absent). Follow `change-api-models` for validators.

## 2. Docs module

Create `app/docs/<name>.py` with an `ApiDocs(request_examples=..., response_examples=...)` and export
it from `app/docs/__init__.py` (import and `__all__`). Document **every** status the route can
return, each with `description`, `model` (`ResponseModel` for the error body) and `content` with an
`example` (or named `examples`). At minimum the success response and 500. Copy the shape of
`app/docs/health.py` (simple) or `app/docs/authenticate.py` (request examples and several errors).
Examples must validate against their model: the tests check.

## 3. Route

In `app/app.py`, next to the others:

```python
@app.get(
    "/foo",
    response_class=JSONResponse,
    responses=foo_docs.response_examples,
    tags=["Monitoring"],
)
async def foo() -> JSONResponse:
    """One-line summary used by Swagger.

    Longer description of what the endpoint does and its parameters.
    """
```

- Raise `PESUAcademyError` subclasses for failures (`add-error-type`); never return error bodies.
- A GET that monitors or scripts may call also gets a HEAD route, stacked above it and kept out of the
  schema, as `/health` has: `@app.head("/foo", include_in_schema=False)`. A protected route repeats
  its `dependencies=` on the HEAD route too.
- Security headers, 404s and 405s are handled for every route already; add nothing for them.
- Timestamps: `datetime.datetime.now(IST).isoformat()`.
- Use the module singletons (`metrics`, `pesu_academy`) by name at call time, never captured.
- Protected endpoint: `dependencies=[Depends(require_metrics_token)]`, or a new dependency built the
  same way (`HTTPBearer(auto_error=False)` + raising a `PESUAcademyError`) so the 401 keeps this
  API's body and is counted.
- Names used in the signature must be importable at runtime (the `TYPE_CHECKING` trap in
  `.agents/pesudev-skills/auth/docs/conventions.md`).

## 4. Metrics

Route-level traffic, status and latency are recorded automatically by the middleware using the route
template. Add domain metrics only if the endpoint has an outcome worth counting (`add-metric`).
Never label by a raw path parameter value.

## 5. Tests

- `tests/unit/test_<name>.py` (or extend `test_app_unit.py`) with a `TestClient(app)` fixture (the
  lifespan makes no upstream calls); cover success, each error, and validation failures (400 with the
  standard body).
- Run `uv run pytest tests/unit/test_openapi_docs.py tests/unit/test_openapi_contract.py -q`: they
  check the route is tagged, has a summary and description, documents success and 500, that examples
  validate, and that every generated request gets a documented response. Fix the docs, not the tests.
  Routes that take no input are also listed in `PAGES` in `tests/unit/conftest.py`.
- If the endpoint calls PESU Academy, add a live test marked `secret_required`.

## 6. Documentation and benchmark

- README: add a row to the endpoints table in "How to use the PESUAuth API" and a `### /foo` section
  (parameters, responses table, response object), in the same style as `/health` and `/metrics`.
  The API Explorer and Swagger UI are built from `/openapi.json`, so they need no change.
- If the endpoint should be benchmarkable, add it to the `--route` choices in
  `scripts/benchmark/benchmark_requests.py`. `make_request` in `scripts/benchmark/util.py` already
  sends a GET for any route other than `authenticate`; a new POST route needs its own branch there.

## 7. Verify

`run-tests`, then `run-locally`: call the endpoint with curl, check the body, status, the API
Explorer at `/` and Swagger at `/docs`, and that `/metrics` shows `route_requests_total{route="/foo"}`.
A new endpoint is a minor version bump (additive).
