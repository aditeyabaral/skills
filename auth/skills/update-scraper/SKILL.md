---
name: update-scraper
description: Change how pesu-auth reads PESU Academy's API - follow a change to the login or profile request or response, or add, change or remove a profile field end to end - using redacted captures, the upstream response models, unit fixtures and one live verification. Use for 422 profile-parse errors, unexpected 502s or 401s, rising missing_field or unexpected_value counts, or any profile-field request.
---

# Updating how PESU Academy is read

Read `.agents/pesudev-skills/auth/docs/upstream.md` first: the flow, where each field comes from,
and the rules for values. The requests are in `app/pesu.py`; the response shapes are the models in
`app/models/upstream.py`. Nothing is scraped from HTML any more; the skill keeps its old name.

## Ground rules

- Use only the `.env` test account, and keep requests to a handful: no loops, no retries.
- Save captured responses **outside the repository**, redacted (names, IDs, emails, phone numbers,
  dates of birth, the access token). Never commit real personal data or paste it into a PR.
- Values are returned as PESU sends them, from one labelled source, or null. No expansion tables, no
  fallback from one block or response to another, no picking IDs by their format. Ask before adding
  any derived value; the accepted ones are the campus code (mapped from the campus name) and the date
  of birth (converted from a timestamp).
- A changed field is null and counted (`missing_field`, `unexpected_value`), not a failed login. Only
  a structural change is a `ProfileParseError` (422, `response_structure`).
- Declare only fields the API returns: the models are what keeps the rest of PESU's response (photos,
  addresses, parents' details, marks) out of logs and memory.

## A. PESU changed its API

1. Identify the failing step: `AuthenticationError` for valid credentials (the login URL, form or
   401 handling), `UpstreamError` (the login answered oddly: read the logged `detail`), or
   `ProfileFetchError`, `ProfileParseError` or the `profile_parse_errors_total` reasons (the profile
   request or its shape). The WARNING logs name the fields that failed, never their values.

1. Capture the current responses with a throwaway script outside the repo that reuses the app's
   request constants, for example:

   ```python
   import asyncio
   import json
   import os

   import httpx2
   from dotenv import load_dotenv

   from app.pesu import DISPATCHER_URL, LOGIN_FORM, LOGIN_URL, MOBILE_HEADERS, PROFILE_FORM, _multipart

   load_dotenv()


   async def main():
       form = {"userName": os.environ["TEST_PRN"], "password": os.environ["TEST_PASSWORD"], **LOGIN_FORM}
       async with httpx2.AsyncClient(timeout=10.0) as client:
           login = await client.post(LOGIN_URL, files=_multipart(form), headers=MOBILE_HEADERS)
           token = login.json()["accessToken"]
           headers = {**MOBILE_HEADERS, "Authorization": f"Bearer {token}"}
           profile = await client.post(DISPATCHER_URL, files=_multipart(PROFILE_FORM), headers=headers)
       # The raw response: real personal data, kept outside the repository
       with open("/path/outside/repo/profile.json", "w") as f:
           json.dump(profile.json(), f, indent=2)


   asyncio.run(main())
   ```

   Run it from the repo root with `PYTHONPATH=. uv run python /path/outside/repo/capture.py`
   (`PYTHONPATH=.` so the script can import `app`). The file holds the account's real data: redact
   it (replace every personal value with an invented one) before reading it into any conversation,
   report or log, and delete the raw copy when you are done.

1. Compare the shape with the models in `app/models/upstream.py`. Change the minimum: a URL, a form
   value, a field alias, a block. Keep the comments that say where each value was observed.

1. Update the fixture payloads in `tests/unit/conftest.py` (`LOGIN_PAYLOAD`, `PROFILE_PAYLOAD`) to the
   new shape, with invented values, and keep tests for each failure path.

## B. Add, change or remove a profile field

The repository's `docs/development.md` ("Adding a profile field") lists every place, and tests fail
when one is missed:

1. The model for its block in `app/models/upstream.py` (from `STUDENT_INFO` if it has the field).
1. `PESUAcademy._build_profile` in `app/pesu.py`.
1. `ProfileField` and `ProfileModel` in `app/models/profile.py`: same position in both, with title,
   description and example; the description becomes the Swagger docs.
1. Both full-profile examples in `app/docs/authenticate.py`, and the README `ProfileObject` table.
1. The fixture payloads and `FULL_PROFILE` in `tests/unit/conftest.py`.
1. A `TEST_*` variable named after the field in `.env.example`, `PROFILE_VARIABLES` in
   `tests/conftest.py` (and `INTEGER_FIELDS` if it is an integer), and the test job's `env` in
   `.github/workflows/ci_checks.yml`. A maintainer has to add the secret: say so in the PR.

A new field behaves like every other: in the default set, filterable, logged with the profile, null
when PESU has no value. Do not add opt-in tiers or redaction for it, and do not add fields that are
not about the student's identity or enrolment (the photo and blood group were left out on purpose).
Removing or renaming a field is breaking: follow section 4 of `change-api-models`.

## C. Verify

```bash
uv run pytest tests/unit/test_pesu.py tests/unit/test_authenticate_flow.py tests/unit/test_metrics_instrumentation.py -q
uv run python scripts/run_tests.py
uv run pytest -m secret_required -v          # once, with credentials
```

If credentials are unavailable, finish everything else and state in the PR that live verification is
outstanding. If PESU is down, stop and report.
