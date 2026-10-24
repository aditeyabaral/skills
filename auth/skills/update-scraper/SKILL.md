---
name: update-scraper
description: Change how pesu-auth reads PESU Academy - fix selectors after PESU changes its pages, adjust the CSRF or login flow, or add, rename or remove a profile field end to end - using redacted HTML, loud parse failures, unit fixtures and one live verification. Use for 422 profile-parse errors, CSRF 502s, unexpected 401s, or any profile-field request.
---

# Updating the PESU Academy scraper

Read `.agents/pesudev-skills/auth/docs/upstream.md` first. All scraping is in `app/pesu.py`.

## Ground rules

- Use only the `.env` test account, **one session at a time**, and keep requests to a handful.
- Save captured HTML **outside the repository**, redacted (replace names, PRN/SRN, emails, phone
  numbers). Never commit real personal data or paste it into a PR.
- Failures stay loud: unknown or missing structure raises `ProfileParseError` and increments
  `PROFILE_PARSE_ERRORS` with a specific `reason`. Do not return a partial profile silently.

## A. PESU changed a page

1. Identify the failing step from the error: `CSRFTokenError` (token tag on `/Academy/` or after
   login), `AuthenticationError` for valid credentials (login marker or form fields), or
   `ProfileParseError` + reason (profile page).

1. Capture the current page with a throwaway script outside the repo that reuses the app's client
   code, for example:

   ```python
   import asyncio
   import os

   from dotenv import load_dotenv
   from app.pesu import PESUAcademy

   load_dotenv()


   async def main():
       pesu = PESUAcademy()
       client, token = await pesu._fetch_new_client_with_csrf_token()
       try:
           r = await client.post(
               "https://www.pesuacademy.com/Academy/j_spring_security_check",
               data={"_csrf": token, "j_username": os.environ["TEST_PRN"], "j_password": os.environ["TEST_PASSWORD"]},
           )
           # fetch the profile page with the same query as get_profile_information, then save a redacted copy
       finally:
           await client.aclose()


   asyncio.run(main())
   ```

   Run it from the repo root with `PYTHONPATH=. uv run python /path/outside/repo/capture.py`
   (`PYTHONPATH=.` so the script can import `app`).

1. Compare the structure with the selectors in `.agents/pesudev-skills/auth/docs/upstream.md`. Change the minimum: a selector,
   `PROFILE_PAGE_HEADER_TO_KEY_MAP`, the node count, or the login-failure marker.

1. Update the unit-test HTML in `tests/unit/test_pesu.py` to the new structure (fake values), and keep
   tests for each failure reason.

## B. Add a profile field

Every one of these must change together:

1. `ProfileField` Literal in `app/pesu.py` (drives `DEFAULT_FIELDS` and request validation).
1. Parsing: a new `PROFILE_PAGE_HEADER_TO_KEY_MAP` entry if it is one of the labelled form groups
   (and revisit the `range(7)` loop and the `< 7` check if the count changes), or a dedicated
   selector like `#updateMail`.
1. `ProfileModel` in `app/models/profile.py`: optional field with title, description, example.
1. `app/docs/authenticate.py`: add it to the full-profile example (and the selected-fields example if
   sensible).
1. README `ProfileObject` table and the Python integration example response.
1. `.env.example` and the `pre-commit.yaml` `env:` block only if a live test needs a new `TEST_*`
   value; a new secret must be added by a maintainer, so escalate.
1. Tests: unit parsing test with fixture HTML, field filtering with the new field, live assertions in
   `tests/functional/test_authenticate_functional.py` and `tests/integration/test_app_integration.py`.

Removing or renaming a field is breaking: follow section 4 of `change-api-models`.

## C. Verify

```bash
uv run pytest tests/unit/test_pesu.py tests/unit/test_metrics_instrumentation.py -q
uv run python scripts/run_tests.py
uv run pytest -m secret_required -v          # once, alone, with credentials
```

If credentials are unavailable, finish everything else and state in the PR that live verification is
outstanding. If PESU is down, stop and report.
