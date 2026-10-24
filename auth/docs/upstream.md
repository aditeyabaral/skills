# Upstream: PESU Academy

pesu-auth signs students in through the API that PESU Academy's mobile app uses. That API is
undocumented: every URL, form value and response key below was read off the app's own traffic, and
PESU can change any of it with an app release, without notice. Most production incidents start here.
The requests are in `app/pesu.py`; the response shapes are pydantic models in
`app/models/upstream.py`.

## The flow

At most two calls per `/authenticate` request, both multipart `POST`s (`files=_multipart(form)`, not
`data=`) carrying `X-Client-Type: MOBILE`:

| Step                  | Operation label | Request                                                                                                                                                                   | Outcome                                                                                                                                                                                                                                          |
| --------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1. Login              | `login`         | `LOGIN_URL` (`https://www.pesuacademy.com/MAcademy/mobile/mobilelogin/auth`), form `userName`, `password` and `LOGIN_FORM` (`j_appId=YES`, `instId=1,6,7,14`)             | HTTP 401: `AuthenticationError`. Any other non-200, a body that is not a `LoginResponse`, or `mobileJsonObject.login` other than `"SUCCESS"`: `UpstreamError` (502). Otherwise `accessToken` is kept for step 2                                  |
| 2. Profile (optional) | `profile_fetch` | `DISPATCHER_URL` (`https://www.pesuacademy.com/MAcademy/mobile/dispatcher`), form `PROFILE_FORM` (`action=27`, `mode=1`, `menuId=11172`), `Authorization: Bearer <token>` | Non-200, PESU's error envelope (`{"status": 4xx, ...}` sent with HTTP 200), or a `MESSAGE` that does not start with `SUCCESS`: `ProfileFetchError` (502). A body that is not a `ProfileResponse`: `ProfileParseError` (422). Otherwise a profile |

A login that succeeds without a usable `accessToken` is fine without a profile and an
`UpstreamError` with one. A connection failure or a timeout (`httpx2.HTTPError`) is an
`UpstreamError` at step 1 and a `ProfileFetchError` at step 2.

Each login gets its own `httpx2.AsyncClient(timeout=UPSTREAM_TIMEOUT_SECONDS)` (10 seconds), used
for its two calls only and closed before it returns. Redirects are not followed (the httpx default):
these endpoints answer directly, so a redirect means something in front of them changed, and it
surfaces as a 502. Nothing is cached or prepared between logins, and the app makes no upstream call
at startup, so it starts without network access to PESU Academy.

## Reading the profile

`ProfileResponse` reads two blocks. Everything else in the response (the photo, blood group,
addresses, parents' details, marks, `STUDENT_SEMESTERS`, `PLACEMENT_DETAILS`, ...) is dropped while
parsing, because the models declare only the fields the service returns (`extra="ignore"`).

| Field                                         | Source                                                                                                             |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `prn`, `srn`                                  | `STUDENT_INFO.LoginId`, `STUDENT_INFO.SRN` (the same ID for students who joined before SRNs existed)               |
| `name`, `firstName`, `middleName`, `lastName` | `STUDENT_INFO.NameAsInSSLC`, `.FirstName`, `.MiddleName`, `.LastName`                                              |
| `email`, `mobile`                             | `STUDENT_INFO.Email`, `.Mobile`                                                                                    |
| `program`, `branch`, `branchShortCode`        | `STUDENT_INFO.ProgramAbbreviation`, `.Branch`, `.BranchAbbreviation`                                               |
| `semester`, `section`                         | `STUDENT_INFO.ClassName`, `.SectionName`, as sent (`"Sem-4"`, `"Section C"`); null once a student graduates        |
| `dateOfBirth`                                 | `STUDENT_INFO.DateOfBirth`: epoch milliseconds of midnight IST, turned into `YYYY-MM-DD` in IST                    |
| `campus`, `gender`                            | `STUDENT_PHOTO.instituteName`, `STUDENT_PHOTO.gender`: the only fields `STUDENT_INFO` lacks                        |
| `campusCode`                                  | Mapped from `campus` by `CAMPUS_CODES`: `PES University (Ring Road)` is 1, `PES University (Electronic City)` is 2 |

The rules, each decided by the maintainers (do not change one without asking):

- **Values are returned as PESU sends them.** Text is stripped; nothing is expanded, reformatted or
  inferred. `program` stays `B.Tech.`; IDs are never picked by their format.
- **One source per field.** Every field `STUDENT_INFO` has comes from `STUDENT_INFO` alone. No other
  block, and never the login response, stands in for a value it lacks.
- **Every requested field is present, null when there is no value.** A blank string or null from PESU
  is null. `"NA"` is *not*: the mobile API does not use it to mark a missing value, so it is returned as
  text. Numbers sent where text is expected (a mobile number) are the same value
  (`coerce_numbers_to_str`).
- **A changed field does not fail the login.** A field PESU stops sending is null and counted as
  `profile_parse_errors_total{reason="missing_field"}`; one sent in an unexpected shape is null and
  counted as `unexpected_value`. Both are counted once per field and logged by PESU's name for the
  field (`STUDENT_INFO.SRN`), never by value. A campus name with no known code returns the campus with
  a null `campusCode` and counts `unknown_campus_code`.
- **Only a structural change is a 422.** A body that is not JSON of the expected shape, or a success
  whose `STUDENT_INFO` holds no student data, raises `ProfileParseError` and counts
  `response_structure`.
- **Never log what failed to parse.** A pydantic `ValidationError` quotes its input, which is a
  response full of personal data. Log `_validation_failure_summary(e)` (locations, types and messages
  only) and raise with `from None`.

## Recognising a PESU-side change

Signals, in the order they usually appear:

- `profile_parse_errors_total{reason}` rising for `missing_field`, `unexpected_value` or
  `response_structure`, or 422s on `/authenticate` with `profile: true`.
- `UpstreamError` or `ProfileFetchError` rising in `errors_total{type}` (502s), with
  `upstream_responses_total{status}` showing what PESU answered.
- `upstream_requests_total{outcome="error"}` rising: PESU is down, slow or blocking us; nothing in
  our code changed.
- Every login failing with 401 while the credentials are right: the login form or URL changed.
- The live tests failing on a field comparison: they compare every field with the account's `TEST_*`
  value, so a mismatch after a PESU app release usually means their API changed.

## Investigating safely

Use only the `.env` test account, a handful of requests, and no loops. Save captured responses
**outside the repository**, redacted (names, IDs, emails, phone numbers, dates of birth, and the
access token). Never commit a real response or paste one into a PR; build test payloads from the
shapes in `tests/unit/conftest.py` with invented values. See the `update-scraper` skill.

## Verifying against the real site

- `uv run pytest -m secret_required -v` with the `TEST_*` credentials: the functional and integration
  live tests log in for real and compare the whole profile.
- Or run the app (`uv run python -m app.app --debug`) and POST to `/authenticate` with the test
  account (the `run-locally` skill).

## Etiquette

PESU Academy is a university system we do not own. Keep request volume minimal: no retries, no
polling, no live tests in a loop. Benchmarks against a local instance still log in to PESU for real;
keep `--num-requests` small.
