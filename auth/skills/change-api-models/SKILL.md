---
name: change-api-models
description: Change pesu-auth request, response or profile models - add, rename, remove or retype fields, add validation - keeping camelCase aliases, strictness, documentation examples, README tables and backwards-compatibility rules in step. Use for any change to what /authenticate (or another route) accepts or returns.
---

# Changing API models

Models: `app/models/request.py` (`RequestModel`), `app/models/response.py` (`ResponseModel`),
`app/models/profile.py` (`ProfileModel`), `app/models/metrics.py` (`MetricsModel`, see `add-metric`).
Contract rules: `.agents/pesudev-skills/auth/docs/api-contract.md`.

## 1. Decide compatibility first

| Change                                                                                                          | Compatible?                                                         |
| --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| New optional request field with a default                                                                       | Yes (minor)                                                         |
| New response or profile field                                                                                   | Yes (minor): callers must tolerate new keys                         |
| New allowed value in a Literal/enum                                                                             | Yes (minor)                                                         |
| Removing/renaming a field, changing a type, making optional required, tightening validation, changing a default | **No (major)**: escalate unless the request explicitly asked for it |

## 2. Model conventions

- Python names snake_case; wire names camelCase via `alias_generator=to_camel`. Never hand-write an
  alias that `to_camel` would produce.
- `strict=True` stays on. If one field genuinely needs lax parsing, relax only that field with
  `Field(..., strict=False)` and a comment explaining why (precedent: `ResponseModel.timestamp`).
- Request models keep `extra="forbid"`, so unknown and removed keys are a 400.
- Every field: `title`, `description`, `json_schema_extra={"example": ...}`.
- Validators are `@field_validator` classmethods with a docstring, raising `ValueError` with a
  message a caller can act on; the 400 message is built from it (`body.<field>: Value error, <msg>`).
- Optional response fields default to `None`. `/authenticate` dumps with `exclude_unset=True`, so a
  field that was set to `None` is returned as `null` and only a field never set (the profile when it
  was not requested, a field filtered out) is absent. Never switch it to `exclude_none`.

## 3. Every place a field appears

For a request field:

- `RequestModel` + validator, with a length limit for any string; `KNOWN_REQUEST_FIELDS` in `app/app.py` (validation-error metric label
  set; add the wire name, or it is counted as `other`).
- Where it is consumed: `authenticate()` in `app/app.py` → `PESUAcademy.authenticate` in `app/pesu.py`.
- `app/docs/authenticate.py` request examples (they are validated against `RequestModel`).
- README "Request Parameters" table, and the `authenticate` route docstring in `app/app.py`.

For a profile field: follow `update-scraper` (it covers `ProfileField` in `app/models/profile.py`,
the upstream model and `_build_profile`).

For a response field:

- `ResponseModel` (or `ProfileModel`), the code that fills it, `app/docs/authenticate.py` response
  examples (validated against `ResponseModel`), README "Response Object"/`ProfileObject` tables and
  the full request and response under "Integrating your application with the PESUAuth API".

## 4. Removing or renaming (breaking)

- Remove it from the model; `extra="forbid"` turns old payloads into a 400 naming the key.
- Add regression tests that send the old key (snake_case and camelCase) and assert the 400 message,
  like `test_validate_deprecated_*` in `tests/unit/test_request_model.py`.
- Major version bump, `feat!:` commit, and a "Breaking change" section in the PR.

## 5. Tests

- `tests/unit/test_request_model.py`: valid values, each invalid value, strictness (wrong types are
  rejected, not coerced), aliasing (camelCase accepted, snake_case rejected where `extra="forbid"`).
- Route-level tests through `TestClient` for the 400 body and message.
- `uv run pytest tests/unit/test_openapi_docs.py tests/unit/test_openapi_contract.py -q` for the
  examples and the schema.
