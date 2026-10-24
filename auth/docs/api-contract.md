# API contract

What callers depend on, and the rules for changing it. The README (`README.md`, section "How to use
the PESUAuth API") is the public statement of this contract, `/openapi.json` (rendered by the API
Explorer at `/` and Swagger UI at `/docs`) is the machine-readable one, and the tests below keep the
two honest. A change to any of them must update the others.

## Endpoints

| Route           | Methods   | Tag             | Success                                                      | Errors                            | Notes                                                                                         |
| --------------- | --------- | --------------- | ------------------------------------------------------------ | --------------------------------- | --------------------------------------------------------------------------------------------- |
| `/`             | GET, HEAD | (not in schema) | 200 HTML: the API Explorer (`app/templates/playground.html`) |                                   | Built from `/openapi.json` in the browser; no build step                                      |
| `/docs`         | GET, HEAD | (not in schema) | 200 Swagger UI                                               |                                   | `docs_url="/docs"`; `/redoc` serves ReDoc                                                     |
| `/openapi.json` | GET, HEAD | (not in schema) | 200                                                          |                                   | Generated, with the overrides described in `.agents/pesudev-skills/auth/docs/architecture.md` |
| `/authenticate` | POST      | Authentication  | 200 `ResponseModel`                                          | 400, 401, 422, 500, 502           | Body `RequestModel`; `Cache-Control: no-store`                                                |
| `/health`       | GET, HEAD | Monitoring      | 200 `{status: true, message: "ok", timestamp}`               | 500                               | Must stay unauthenticated: uptime monitors and Render call it                                 |
| `/metrics`      | GET, HEAD | Monitoring      | 200 Prometheus text (default) or JSON (`?fmt=json`)          | 400 (bad `fmt`), 401 (token), 500 | Protected only when `METRICS_TOKEN` is set, for HEAD as for GET                               |
| `/readme`       | GET, HEAD | Documentation   | 308 to `https://github.com/pesu-dev/auth`                    | 500                               |                                                                                               |

HEAD routes are separate `@app.head(..., include_in_schema=False)` routes stacked on the GET ones,
so each operation is documented once. Every response carries `X-Content-Type-Options: nosniff`,
`Referrer-Policy: no-referrer` and `X-Frame-Options: DENY`.

## `/authenticate`

Request (`app/models/request.py`, `strict=True`, `extra="forbid"`, camelCase aliases):

| Field      | Type                         | Default           | Rules                                                                                                                                   |
| ---------- | ---------------------------- | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `username` | str                          | required          | At most 96 characters; stripped; empty is a 400; only visible characters (no line breaks, since it is logged). SRN, PRN, email or phone |
| `password` | str                          | required          | At most 128 characters, counted as typed (not stripped); blank is a 400                                                                 |
| `profile`  | bool                         | `false`           | Strict bool: `"true"` is a 400                                                                                                          |
| `fields`   | `list[ProfileField]` or null | null (all fields) | Non-empty; each name in `ProfileField` (`app/models/profile.py`); only with `profile: true`, otherwise a 400; repeats dropped           |

Both strings must be valid text: an unpaired surrogate is a 400, not a 500 while the login is built.

Response (`app/models/response.py`): `status`, `message`, `timestamp` (ISO 8601 in IST, `+05:30`),
and `profile` (`app/models/profile.py`) only when it was requested and the login succeeded. The route
dumps with `exclude_unset=True`, so:

- **every requested profile field is present**, and is `null` when PESU has no value for it, does not
  send it, or sends it in an unexpected shape;
- fields the caller filtered out, and `profile` when it was not requested, are absent;
- fields come back in `ProfileModel`'s order, whatever order they were asked for in.

The 17 profile fields, in order: `name`, `prn`, `srn`, `program`, `branch`, `semester`, `section`,
`email`, `mobile`, `campusCode` (1 or 2, or null), `campus`, `firstName`, `middleName`, `lastName`,
`branchShortCode`, `gender`, `dateOfBirth`. Where each comes from is in
`.agents/pesudev-skills/auth/docs/upstream.md`. `ProfileField`, `ProfileModel`,
`PESUAcademy._build_profile`, the README `ProfileObject` table, the full-profile examples in
`app/docs/authenticate.py`, `FULL_PROFILE` in `tests/unit/conftest.py` and `PROFILE_VARIABLES` in
`tests/conftest.py` must all agree; tests fail when one of them misses a field.

## Error body and status codes

Every error (4xx and 5xx) is rendered by one of this API's exception handlers as:

```json
{"status": false, "message": "<fixed, documented text>", "timestamp": "2024-07-28T22:30:10.103368+05:30"}
```

| Code | Meaning                                                            | Produced by                                                                                                     |
| ---- | ------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| 400  | The body or query failed validation, or the body could not be read | `validation_exception_handler` (`Could not validate request data - <loc>: <msg>; ...`), `RequestBodyParseError` |
| 401  | Wrong credentials, or a missing or wrong metrics token             | `AuthenticationError`, `MetricsAuthorizationError` (+ `WWW-Authenticate: Bearer`)                               |
| 404  | No route serves the path                                           | `RouteNotFoundError`, from `http_error_handler`                                                                 |
| 405  | The path is not served for the method                              | `MethodNotAllowedError` (+ an `Allow` header naming every method the path serves)                               |
| 422  | The profile response could not be parsed: PESU's API changed       | `ProfileParseError`                                                                                             |
| 500  | Our bug, or a result that failed `ResponseModel` validation        | the catch-all handler, or the route's own `PESUAcademyError(status_code=500)`; generic message                  |
| 502  | PESU Academy unreachable, timed out, declined, or answered oddly   | `UpstreamError` (the login), `ProfileFetchError` (the profile)                                                  |

The message is part of the contract: it is the exception's fixed default. Specifics, such as the
username or what PESU answered, go in the exception's `detail`, which only the log sees. This API
never returns FastAPI's default 422 validation body or Starlette's `{"detail": ...}`; do not
reintroduce either.

## How the contract is enforced

- `tests/unit/test_openapi_docs.py`: every route in the schema is documented, tagged, has a summary
  and a description (its docstring), documents a success and a 500, and has an example and a schema
  for every response; examples validate against the models they name; request examples cover the
  username forms and field filtering; the documented 400, 401, 200, 308 and 502 examples match what
  the running app returns; no phantom `HTTPValidationError` 422 appears and the real 422 is kept; the
  documented full profiles have every field and show nulls.
- `tests/unit/test_openapi_contract.py`: Schemathesis sends generated requests, valid and invalid, to
  the app in process and checks every response's status, content type and body against the schema.
- `tests/unit/test_docs.py` and `test_the_metrics_doc_describes_every_family`: the error table in
  `docs/architecture.md`, the configuration table in `docs/development.md` and the metric rows in
  `docs/metrics.md` list everything the code has.

So a new route needs an `ApiDocs` module in `app/docs/`, exported from `app/docs/__init__.py`, passed
as `responses=` (and `openapi_extra=` for request examples), plus `tags=[...]` and a docstring. See
the `add-endpoint` skill.

## Compatibility and versioning policy

The project version in `pyproject.toml` is the API version, published in `/openapi.json`
`info.version`. Every pull request raises it by **exactly one step** from `dev`'s
(`.github/scripts/check_version_bump.py`, run by the *Version check* job):

- **Minor** (`x.Y+1.0`): the default, whatever the change.
- **Patch** (`x.y.Z+1`): anything (feature, fix, docs or tests) that was meant to be part of the last
  minor release but was missed or split out into its own pull request.
- **Major** (`X+1.0.0`): anything backwards-incompatible: removing or renaming a request or response
  field, changing a type, changing a status code a caller could branch on, rejecting input that used
  to be accepted, or changing the default of `fmt`. Mark the commit `feat!:`/`fix!:` and call it out
  in the PR. Precedents: v3.0.0 moved response keys to camelCase (#151); v4.0.0 removed "Know Your
  Class and Section" (#155); v5.0.0 moved to the mobile API, renamed `phone` to `mobile`, and made
  missing values null instead of absent (#241).

Additive changes are compatible: a new optional request field, a new response or profile field
(callers must tolerate unknown keys), a new route, a new metric.

## Deprecation pattern

When a field is removed, `extra="forbid"` makes old payloads fail loudly with a 400 naming the key,
instead of being silently ignored. Keep regression tests that send the removed keys and assert the
400, like `test_validate_deprecated_*` and `test_validate_removed_kycas_fields_rejected` in
`tests/unit/test_request_model.py` and `test_integration_authenticate_removed_kycas_fields_rejected`
in `tests/integration/test_app_integration.py`. Removing a profile field from `ProfileField` likewise
turns requests for it into a 400 (`test_validate_deprecated_campus_code_in_fields_rejected`); add a
test for that too.
