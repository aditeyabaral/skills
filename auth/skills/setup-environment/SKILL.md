---
name: setup-environment
description: Set up or repair a pesu-auth development environment - Python 3.14 via uv, dependencies, .env test credentials, pre-commit hooks, git remotes - and confirm it works with a baseline test run. Use on a fresh clone, a broken environment, or before starting any change.
---

# Setting up pesu-auth

Run everything from the repository root.

## 1. Check the checkout

```bash
git status                       # clean? which branch?
git remote -v                    # origin = your fork, upstream = pesu-dev/auth
git submodule update --init      # agent material in .agents/pesudev-skills
```

If there is no `upstream` remote: `git remote add upstream https://github.com/pesu-dev/auth`.
Then `git fetch upstream`.

## 2. Install

```bash
uv --version                     # install uv if missing: https://docs.astral.sh/uv/
uv sync --all-groups             # creates .venv with Python 3.14 and the dev group
```

`uv sync` downloads Python 3.14 itself if needed (`requires-python = ">=3.14"`). Do not use
`pip install -r requirements.txt`: that file is stale (see `.agents/pesudev-skills/auth/docs/known-issues.md`).

## 3. Credentials for live tests (optional)

```bash
test -f .env || cp .env.example .env
```

The `TEST_*` variables are one real PESU test account. If the human has not provided them, leave the
placeholders: the suite runs without them and reports that live tests were skipped. Never print or
commit `.env` (`.gitignore` already excludes it). `METRICS_TOKEN` should normally **not** be set in
`.env`; tests pin it off anyway.

## 4. Hooks

```bash
uv run pre-commit install
```

## 5. Baseline

```bash
uv run pytest tests/unit -q                 # offline, a few seconds
uv run python scripts/run_tests.py          # full run with coverage gate
```

Record the counts and coverage (today: 265 passed, 11 live deselected without credentials, 100%).
If the baseline fails before you changed anything, stop and report it: the problem is not yours to
hide inside another change.

## Troubleshooting

| Symptom                                                     | Fix                                                                           |
| ----------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `No interpreter found for Python >=3.14`                    | `uv python install 3.14`                                                      |
| Import errors for `httpx2`/`selectolax`                     | `uv sync --all-groups` (you are outside the venv or it is stale)              |
| Integration or functional tests fail with connection errors | No internet access to pesuacademy.com; run `tests/unit` only and say so       |
| Many `/metrics` tests fail with 401                         | A `METRICS_TOKEN` is exported in your shell; `unset METRICS_TOKEN`            |
| Live tests fail with 401 on valid credentials               | Another session is using the test account (CI or another run); wait, run once |
| `pre-commit` hook environments fail to build                | `uv run pre-commit clean && uv run pre-commit install`                        |
