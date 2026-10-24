---
name: run-tests
description: Run the pesu-auth test suites correctly - targeted pytest, the canonical scripts/run_tests.py with its 100% line and branch coverage gate, the live tests against PESU Academy, and the full pre-commit run - and report results truthfully, including partial runs. Use whenever you need to know if the code works.
---

# Running tests

## Commands

| Goal                             | Command                                                                            |
| -------------------------------- | ---------------------------------------------------------------------------------- |
| One test                         | `uv run pytest tests/unit/test_pesu.py::test_name -v`                              |
| One file / keyword               | `uv run pytest tests/unit/test_request_model.py -q` / `uv run pytest -k fields -q` |
| Offline suite                    | `uv run pytest tests/unit -q`                                                      |
| Everything without credentials   | `uv run pytest -m "not secret_required" -q` (needs internet)                       |
| **Canonical run** (what CI runs) | `uv run python scripts/run_tests.py`                                               |
| Live tests only                  | `uv run pytest -m secret_required -v`                                              |
| All hooks incl. tests            | `uv run pre-commit run --all-files`                                                |

`scripts/run_tests.py` loads `.env` and runs `pytest --cov --cov-report=term-missing --disable-warnings -v -s`, with coverage configured in `pyproject.toml` (`app/`, `scripts/`,
`.github/scripts/`, branches counted, fail under 100%). It deselects the live tests and warns
**"Live PESU tests skipped"** unless `TEST_EMAIL`, `TEST_PRN`, `TEST_SRN`, `TEST_MOBILE` and
`TEST_PASSWORD` are all set. The tests need the `benchmark` group too (`uv sync --all-groups`).

## The live tests

Every live run makes real logins to PESU Academy. Before one:

- run it because the change needs it (`app/pesu.py`, `app/models/upstream.py`, `/authenticate`),
  once, not in a loop;
- make sure no other live run or benchmark (yours, another agent's, a terminal) is in progress;
- never pass `-n`/`--numprocesses`.

A live test failing on a field comparison after a PESU app release usually means PESU's API changed
(`update-scraper`); a 401 on credentials that worked before means the account's credentials changed
or PESU's login did. Rerun **once** at most before reporting.

## Interpreting results

- **Coverage:** the gate is 100% of lines and branches. Add tests for the lines and branches listed
  under `Missing`; never exclude code to pass.
- **Ordering:** `tests/conftest.py` forces unit → functional → integration. A test that passes alone
  but fails in the suite usually leaks state through a module singleton (`app.app.metrics`,
  `app.app.pesu_academy`, `app.openapi_schema`). Fix the isolation, not the order.
- **Network failures** in functional or integration tests (connection errors, timeouts): PESU
  Academy or the network is down. Not a code failure; say so and rely on `tests/unit` plus a later
  rerun.
- **One `DeprecationWarning`** from Starlette's test client is expected.

## Reporting

Always report the exact command and its summary line, for example:

> `uv run python scripts/run_tests.py` → 640 passed, 14 deselected, coverage 100%. **Partial run:**
> live PESU tests skipped (no credentials).

Never describe a partial run as a full pass, never omit failures, and never paste credential values
or personal data from live-test output.
