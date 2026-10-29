---
name: navigate-codebase
description: Find where things happen in pesu-auth and understand a flow before changing it - module map, entry points, where each concern lives, search recipes, and the reading order for common tasks. Use when you need to locate code, trace a request, or answer a question about how the service works.
---

# Navigating pesu-auth

The codebase is small; read whole files rather than guessing from fragments. Background: the
repository's `docs/architecture.md`, then `.agents/pesudev-skills/auth/docs/architecture.md`.

## Where things live

| Question                                     | Look in                                                                                                                          |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| What routes exist and what do they do?       | `app/app.py` (search `@app.`)                                                                                                    |
| How is a login performed?                    | `PESUAcademy.authenticate` in `app/pesu.py`                                                                                      |
| How is the profile fetched and built?        | `PESUAcademy._fetch_profile` and `_build_profile` in `app/pesu.py`; the shapes in `app/models/upstream.py`                       |
| Which profile fields exist?                  | `ProfileField` and `ProfileModel` in `app/models/profile.py`                                                                     |
| What does the request accept?                | `RequestModel` in `app/models/request.py`                                                                                        |
| Which status code does an error produce?     | The exception class in `app/exceptions/`; handlers in `app/app.py`                                                               |
| What metrics exist?                          | `FAMILIES` in `app/metrics/collector.py`; `docs/metrics.md`                                                                      |
| Where is a metric recorded?                  | `rg -n "<CONSTANT_NAME>" app`                                                                                                    |
| What do Swagger and the Explorer show?       | `app/docs/<route>.py`, route decorators in `app/app.py`; the Explorer is `app/templates/playground.html` (`change-api-explorer`) |
| How are tests run in CI?                     | The *Test suite & coverage* job in `.github/workflows/ci_checks.yml`, `scripts/run_tests.py`                                     |
| What does CI check?                          | `.github/workflows/ci_checks.yml`, `.agents/pesudev-skills/auth/docs/ci-cd.md`                                                   |
| Where do Copilot's custom agents come from?  | Generated into `.github/agents/` from `.agents/pesudev-skills/auth/agents/` by `scripts/sync_agents.py`                          |
| How is the image built?                      | `Dockerfile`                                                                                                                     |
| Contribution rules                           | `.github/CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`                                                                    |
| How to develop, test and deploy (for humans) | `docs/` (`development.md`, `testing.md`, `deployment.md`)                                                                        |

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
  `tests/unit/test_authenticate_flow.py`, `tests/unit/test_pesu.py`, `tests/integration/test_app_integration.py`.
- **PESU Academy and the profile:** `.agents/pesudev-skills/auth/docs/upstream.md` → `app/pesu.py`
  (`_login`, `_fetch_profile`, `_build_profile`) → `app/models/upstream.py` → the payload fixtures in
  `tests/unit/conftest.py` → `tests/unit/test_pesu.py` → `tests/functional/`.
- **Metrics:** `docs/metrics.md` and `.agents/pesudev-skills/auth/docs/metrics.md` → `app/metrics/collector.py` → the recording site → `app/models/metrics.py`
  → `app/docs/metrics.py` → `tests/unit/test_metrics_*.py`.
- **Errors:** `app/exceptions/` → handlers in `app/app.py` → `tests/unit/test_app_unit.py`,
  `tests/unit/test_authenticate_flow.py`.
- **Concurrency/lifecycle:** `.agents/pesudev-skills/auth/docs/architecture.md` (the upstream client) → `app/pesu.py` top to bottom →
  the `*closed*`, `*cancel*` and `*concurrent*` tests in `tests/unit/test_pesu.py`.

## Checking your understanding

Before changing a flow, run it. Start the app (`run-locally` skill) and send the request, or write a
one-off `TestClient` snippet in a scratch file outside the repo. Confirm the behaviour you are about
to change is the behaviour you think it is.
