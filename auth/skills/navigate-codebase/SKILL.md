---
name: navigate-codebase
description: Find where things happen in pesu-auth and understand a flow before changing it - module map, entry points, where each concern lives, search recipes, and the reading order for common tasks. Use when you need to locate code, trace a request, or answer a question about how the service works.
---

# Navigating pesu-auth

The codebase is small (about 700 statements in `app/`); read whole files rather than guessing from
fragments. Background: `.agents/pesudev-skills/auth/docs/architecture.md`.

## Where things live

| Question                                    | Look in                                                                                                 |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| What routes exist and what do they do?      | `app/app.py` (search `@app.`)                                                                           |
| How is a login performed?                   | `PESUAcademy.authenticate` in `app/pesu.py`                                                             |
| How is the profile scraped?                 | `PESUAcademy.get_profile_information`, `_extract_and_update_profile`, `PROFILE_PAGE_HEADER_TO_KEY_MAP`  |
| Which profile fields exist?                 | `ProfileField` in `app/pesu.py`, `ProfileModel` in `app/models/profile.py`                              |
| What does the request accept?               | `RequestModel` in `app/models/request.py`                                                               |
| Which status code does an error produce?    | The exception class in `app/exceptions/`; handlers in `app/app.py`                                      |
| What metrics exist?                         | `FAMILIES` in `app/metrics/collector.py`; README "`/metrics`"                                           |
| Where is a metric recorded?                 | `rg -n "<CONSTANT_NAME>" app`                                                                           |
| What does Swagger show?                     | `app/docs/<route>.py`, route decorators in `app/app.py`                                                 |
| How are tests run in CI?                    | `.pre-commit-config.yaml` (the `pytest` hook), `scripts/run_tests.py`                                   |
| What does CI check?                         | `.github/workflows/*.yaml`, `.agents/pesudev-skills/auth/docs/ci-cd.md`                                 |
| Where do Copilot's custom agents come from? | Generated into `.github/agents/` from `.agents/pesudev-skills/auth/agents/` by `scripts/sync_agents.py` |
| How is the image built?                     | `Dockerfile`                                                                                            |
| Contribution rules                          | `.github/CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`                                           |

## Search recipes

```bash
rg -n "def |class " app/pesu.py            # outline of a module
rg -n "ProfileParseError" app tests         # every raise and every test of an error
rg -n "campusCode|campus_code" -g '!uv.lock' # a field across code, docs and tests
rg -n "@pytest.mark.secret_required" tests  # the live tests
git log --oneline -- app/pesu.py | head     # history of a file
git log -S "compare_digest" --oneline       # when a construct was introduced
git show <sha>                              # the reasoning: commit bodies are detailed
```

Commit messages in this repo explain *why* in depth. When code looks strange, `git log -L` or
`git blame` on the lines, then read the commit body before changing it.

## Reading order by task

- **Any `/authenticate` change:** `app/models/request.py` → `app/app.py::authenticate` →
  `app/pesu.py::authenticate` → `app/models/response.py` → `app/docs/authenticate.py` → tests
  `tests/unit/test_app_unit.py`, `tests/unit/test_pesu.py`, `tests/integration/test_app_integration.py`.
- **Scraping:** `.agents/pesudev-skills/auth/docs/upstream.md` → `app/pesu.py::get_profile_information` → `tests/unit/test_pesu.py`
  (HTML fixtures) → `tests/functional/`.
- **Metrics:** `.agents/pesudev-skills/auth/docs/metrics.md` → `app/metrics/collector.py` → the recording site → `app/models/metrics.py`
  → `app/docs/metrics.py` → `tests/unit/test_metrics_*.py`.
- **Errors:** `app/exceptions/` → handlers in `app/app.py` → `tests/unit/test_app_unit.py`.
- **Concurrency/lifecycle:** `.agents/pesudev-skills/auth/docs/architecture.md` (client lifecycle) → `app/pesu.py` top to bottom →
  `lifespan` in `app/app.py` → the `*closes*`/`*prefetch*` tests in `tests/unit/test_pesu.py`.

## Checking your understanding

Before changing a flow, run it. Start the app (`run-locally` skill) and send the request, or write a
one-off `TestClient` snippet in a scratch file outside the repo. Confirm the behaviour you are about
to change is the behaviour you think it is.
