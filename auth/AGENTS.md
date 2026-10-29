# pesu-auth

Instructions for coding agents working in the pesu-auth repository (`pesu-dev/auth`). Read this
file first; it tells you where everything else is.

pesu-auth is a small, stateless FastAPI service that authenticates PESU students against PESU
Academy (through the API PESU Academy's mobile app uses) and optionally returns their profile. It
has no database; PESU Academy is its only dependency. It is public, used by other student projects,
and deployed on Render: production `https://pesuauth.onrender.com`, staging
`https://pesuauth-dev.onrender.com`.

## Where the agent material lives

Everything below is in the `.agents/pesudev-skills` git submodule (the pesu-dev/skills repository).
If `.agents/pesudev-skills` is empty, run `git submodule update --init` first.

| What                    | Path                                           | Use                                              |
| ----------------------- | ---------------------------------------------- | ------------------------------------------------ |
| Skills (task playbooks) | `.agents/skills/<name>/SKILL.md`               | Load the one that matches the task (index below) |
| Reference docs          | `.agents/pesudev-skills/auth/docs/<name>.md`   | Deep background, read on demand                  |
| Roles                   | `.agents/pesudev-skills/auth/agents/<role>.md` | Personas for each phase of autonomous work       |

This file, the skills and the roles always name the reference docs in full
(`.agents/pesudev-skills/auth/docs/<name>.md`). Every other path is relative to the repository
root, so `docs/<name>.md` is the repository's own documentation, not a reference doc.

Never edit `AGENTS.md` or `.agents/` in this repository. They come from
pesu-dev/skills; changes go there as a PR. Never edit `.github/agents/` by hand either: it is
generated from the roles by `uv run python scripts/sync_agents.py`, and a pre-commit hook fails when
it is out of date.

## Quick reference

```bash
uv sync --all-groups                         # install everything (Python 3.14, via uv)
cp .env.example .env                         # then fill TEST_* for live tests (never commit .env)
uv run pre-commit install                    # hooks run on commit
uv run python -m app.app --debug             # http://localhost:5000: API Explorer at /, Swagger at /docs
uv run python scripts/run_tests.py           # canonical test run, 100% line and branch coverage gate
uv run pytest tests/unit -q                  # fast offline tests
uv run pre-commit run --all-files            # everything CI runs
uv run ruff check . && uv run ruff format --check .
docker build . --tag pesu-auth && docker run --rm -p 5000:5000 pesu-auth
```

## Repository map

```
app/app.py              FastAPI app: lifespan, routes, exception handlers, middleware, OpenAPI override, CLI
app/pesu.py             PESUAcademy: login and profile calls to PESU Academy, building the profile, client lifecycle
app/models/             RequestModel, ResponseModel, ProfileModel, MetricsModel, HealthModel (strict, camelCase);
                        upstream.py: the shapes read from PESU Academy
app/exceptions/         PESUAcademyError and subclasses (status code and fixed message live on the exception)
app/metrics/            collector (families), middleware, Prometheus renderer, METRICS_TOKEN auth
app/docs/               OpenAPI request/response examples per route (enforced by tests)
app/templates/          playground.html, the API Explorer served at /
docs/                   the repository's own docs (architecture, development, testing, metrics,
                        deployment), kept current by tests/unit/test_docs.py
tests/unit/             offline tests, one file per concern; fixtures in tests/unit/conftest.py
tests/functional/       PESUAcademy against the real site
tests/integration/      whole app via TestClient, real lifespan
scripts/run_tests.py    test runner used by pre-commit and CI
scripts/sync_agents.py  generates .github/agents/ (Copilot custom agents) from the roles
scripts/benchmark/      load benchmarks and their analysis
.github/                workflows (CI Checks, deploys), PR/issue templates, CONTRIBUTING, version-check script
.github/agents/         generated Copilot custom agents, one per role (never edit by hand)
```

Details: `.agents/pesudev-skills/auth/docs/architecture.md`.

## Rules that always apply

1. **Never log, print, store or echo a password, a token or new personal data.** Not in logs,
   exceptions, metrics labels, test output, commits, PRs or reports. The username and the whole
   profile (gender and date of birth included) are logged at INFO by design, for tracing; that is
   not a defect. (`.agents/pesudev-skills/auth/docs/security.md`)
1. **Errors are exceptions.** Raise a `PESUAcademyError` subclass with its fixed `message`, and put
   the specifics in `detail=`, which only the log sees; the handlers render
   `{status, message, timestamp}` for every error, 404 and 405 included. Validation failures are 400,
   never 422 (422 means PESU's profile response could not be parsed).
1. **The API contract is camelCase, strict and documented.** Models use `alias_generator=to_camel`,
   `strict=True`, and request models `extra="forbid"`. Every route has tags, a docstring and an
   `app/docs/` module; `tests/unit/test_openapi_docs.py` enforces it. (`.agents/pesudev-skills/auth/docs/api-contract.md`)
1. **Every upstream call** goes through `_upstream_call` and every `httpx2.AsyncClient` is closed on
   every path, including cancellation. (`.agents/pesudev-skills/auth/docs/architecture.md`)
1. **Metric labels are bounded** and each fact is recorded in exactly one layer. (`.agents/pesudev-skills/auth/docs/metrics.md`)
1. **Code style is enforced**: Google docstrings and full annotations on everything in `app/`,
   ruff with line length 120, comments explain *why*. (`.agents/pesudev-skills/auth/docs/conventions.md`)
1. **Profile values are returned as PESU sends them, or null.** Every requested field is present;
   no inference, expansion or fallback between sources. (`.agents/pesudev-skills/auth/docs/upstream.md`)
1. **Tests**: add tests with every change; the gate is 100% line and branch coverage. Follow the
   fixture recipes in `.agents/pesudev-skills/auth/docs/testing.md`.
1. **Live tests make real PESU logins**: run them when a change needs them, once, never in a loop,
   never alongside another live run or a benchmark, never with `-n`. A run without credentials is
   partial and must be reported as partial.
1. **Every PR raises `version` in `pyproject.toml` by exactly one step** and runs `uv lock`: minor by
   default, patch for work meant for the last minor that was missed or split out, major for breaking
   changes.
1. **Docs move with behaviour**: README, `docs/`, `app/docs/`, CONTRIBUTING and comments are updated
   in the same PR as the change they describe.

## Contribution workflow

Fork → branch from `upstream/dev` → change + tests → pre-commit green → version bump → PR to
`pesu-dev/auth` **`dev`** from the fork (never from `main`, never to `main`). Conventional Commits
(`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`, `!` for breaking) with bodies that explain
why. Merges to `dev` deploy to staging; production is deployed manually by maintainers.
(`.agents/pesudev-skills/auth/docs/ci-cd.md`)

## Working autonomously

You may take a task all the way to an **open pull request** without a human. You may not merge,
push to `dev`/`main` or `pesu-dev/auth`, trigger deploys, change repository settings, or weaken
checks. Stop and hand back when a change is breaking or security-sensitive beyond what was asked,
needs new secrets or infrastructure, or a check still fails after three attempts. Start with the
`implement-feature` skill; the full lifecycle, definition of done and report format are in
`.agents/pesudev-skills/auth/docs/autonomous-workflow.md`. As GitHub Copilot's cloud agent, read
its "Copilot's cloud agent" section first: that agent opens its own pull request and runs behind a
firewall.

## Skills

Each skill is `.agents/skills/<name>/SKILL.md`. If your tool loads skills from `.agents/skills`, use
them that way; otherwise, when a task matches a row below, open that `SKILL.md` and follow it.

| Skill                    | Use when                                                                   |
| ------------------------ | -------------------------------------------------------------------------- |
| `setup-environment`      | First time in a clone, or the environment is broken                        |
| `navigate-codebase`      | You need to find where something happens or understand a flow              |
| `triage-issue`           | Starting from a GitHub issue or a vague request                            |
| `implement-feature`      | Any change end to end: the orchestrator for everything below               |
| `fix-bug`                | Something behaves wrongly; reproduce, test, fix                            |
| `refactor`               | Restructuring without behaviour change                                     |
| `add-endpoint`           | A new route                                                                |
| `change-api-models`      | Changing request, response or profile fields or validation                 |
| `update-scraper`         | PESU Academy's API changed, or a profile field is added or changed         |
| `add-metric`             | A new counter, gauge or summary                                            |
| `add-error-type`         | A new failure mode that needs its own status or message                    |
| `change-upstream-client` | Anything touching the HTTP client, its lifecycle, timeouts or cancellation |
| `update-dependencies`    | Adding, upgrading or removing packages, Python or tooling versions         |
| `update-docs`            | README, `docs/`, OpenAPI docs, CONTRIBUTING, docstrings                    |
| `write-tests`            | Adding or fixing tests                                                     |
| `run-tests`              | Running the suite and reporting results truthfully                         |
| `run-locally`            | Running the app or the Docker image and smoke-testing endpoints            |
| `benchmark`              | Measuring latency or throughput against a running instance                 |
| `review-change`          | Reviewing a diff (yours or someone else's)                                 |
| `security-review`        | Any change to input handling, logging, auth, upstream, dependencies        |
| `bump-version`           | Preparing the version bump for a PR                                        |
| `open-pull-request`      | Committing, pushing to the fork and opening the PR                         |
| `fix-ci`                 | A GitHub Actions check failed                                              |
| `diagnose-production`    | Production or staging misbehaves; reading `/metrics` and status pages      |

## Roles

In `.agents/pesudev-skills/auth/agents/`. Start each as a sub-agent if your tool supports it;
otherwise follow the file yourself for that phase. In GitHub Copilot each role is also a custom
agent (`.github/agents/<role>.agent.md`), selectable from the agent picker.

| Role                    | Does                                                                     |
| ----------------------- | ------------------------------------------------------------------------ |
| `planner`               | Turns a request into acceptance criteria and a concrete plan (read-only) |
| `implementer`           | Writes the code to the plan, test-first                                  |
| `test-engineer`         | Designs and writes tests, runs the suites, owns coverage                 |
| `reviewer`              | Independent correctness and quality review of the diff (read-only)       |
| `security-auditor`      | Security and privacy review of the diff (read-only)                      |
| `docs-writer`           | Keeps README, `docs/`, OpenAPI docs and comments in step with behaviour  |
| `release-manager`       | Version bump, commits, push to fork, PR                                  |
| `ci-investigator`       | Diagnoses failing checks and proposes or applies fixes                   |
| `scraper-specialist`    | Anything involving PESU Academy's API: login, profile responses, fields  |
| `incident-investigator` | Diagnoses production/staging problems from metrics and logs (read-only)  |

## Reference docs

In `.agents/pesudev-skills/auth/docs/`: `architecture.md`, `conventions.md`, `testing.md`,
`api-contract.md`, `metrics.md`, `upstream.md`, `ci-cd.md`, `security.md`,
`autonomous-workflow.md`, `known-issues.md` (read this before trusting something odd in the repo).
