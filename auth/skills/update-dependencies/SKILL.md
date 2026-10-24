---
name: update-dependencies
description: Add, upgrade or remove pesu-auth dependencies, dev and benchmark tools, pre-commit hooks, the Python version or base images with uv, keeping uv.lock, pyproject.toml, the Dockerfile, CI and pre-commit consistent and verified. Use for dependency bumps, security patches, new packages or toolchain upgrades.
---

# Updating dependencies

Runtime dependencies are in `[project].dependencies`; development tools in
`[dependency-groups].dev`; packages only the benchmark scripts use (`matplotlib`, `numpy`, `pandas`,
`seaborn`, `tqdm`) in `[dependency-groups].benchmark`. All are locked in `uv.lock`. The Docker image
installs from the lock with `uv sync --locked --no-dev`, and every CI job installs with
`uv sync --locked`, which fails on a lock that no longer matches `pyproject.toml` instead of
installing it silently. The lock is what production runs.

## Adding a package

Justify it first: can the standard library or an existing dependency do it? New runtime
dependencies need maintainer agreement (escalate unless requested).

```bash
uv add "<package>>=<version>"                    # runtime
uv add --group dev "<package>>=<version>"        # development tools
uv add --group benchmark "<package>>=<version>"  # used only by scripts/benchmark
```

A package needed on one platform only gets an environment marker, as `tzdata` has
(`"tzdata>=2026.4 ; sys_platform == 'win32'"`): check with `uv export --frozen --no-dev` that it does
not reach the Linux image.

## Upgrading

```bash
uv lock --upgrade-package <package>      # one package
uv lock --upgrade                        # everything (separate PR, larger review)
uv sync --all-groups
```

Raise the lower bound in `pyproject.toml` when the code starts relying on the new version (the
project pins `>=` floors, and past bumps raised them together). Read the changelogs of `fastapi`,
`starlette`, `pydantic`, `httpx2` and `uvicorn` for behaviour changes; note anything relevant in the
PR.

Known sensitive points when upgrading:

- **FastAPI/Starlette:** the OpenAPI override (`_openapi_without_phantom_validation_errors` and
  `_restore_documented_examples`), exception middleware ordering, `HTTPException` handling for 404,
  405 and unreadable bodies, `HTTPBearer(auto_error=False)`, `TestClient`. The OpenAPI, contract and
  metrics tests catch most regressions.
- **pydantic:** strict mode, alias behaviour, wrap validators (`app/models/upstream.py`) and error
  message text (the 400 message is built from it and some tests assert it).
- **httpx2:** client lifecycle, multipart encoding (`files=`), redirect defaults and exception types.
  Run the live tests once after upgrading it.
- **ruff:** pre-commit runs it from `uv.lock` (`uv run --frozen ruff`), so upgrading the `ruff` dev
  dependency upgrades the hook too; there is no separate `rev`. Run `uv run ruff check .` and fix new
  findings in the same PR.
- **schemathesis:** pinned below 5 (`<5`) in `pyproject.toml`; moving to a new major is its own
  change, with `tests/unit/test_openapi_contract.py` as the check.

Relock with `uv lock`, never by editing `uv.lock`. If it rewrites the `revision` line or unrelated
entries, your uv is older than CI's (setup-uv installs the latest): run `uv self update` first.

## Pre-commit hooks

The hooks from other repositories (`pre-commit-hooks`, `mdformat`) are pinned by `rev`:

```bash
uv run pre-commit autoupdate
uv run pre-commit run --all-files
```

## Python version

The floor is in `requires-python`, ruff's `target-version`, both `FROM python:3.14-slim-trixie`
stages of the `Dockerfile`, the `python-version` of `astral-sh/setup-uv` in
`.github/workflows/ci_checks.yml` and of `actions/setup-python` in `copilot-setup-steps.yml`, and the
prerequisites in the README, CONTRIBUTING and `docs/development.md`. Change all of them together.
Raising the floor is a breaking change for anyone running from source: major version.

## Verify

```bash
uv lock --check
uv run python scripts/run_tests.py
uv run pre-commit run --all-files
docker build . --tag pesu-auth-test             # the image must build from the lock
uvx --python 3.14 pip-audit --strict --require-hashes --disable-pip -r <(uv export --frozen --no-dev --no-emit-project)
```

Then the Docker steps from `run-locally`, and one live run if `httpx2`, `pydantic` or the HTTP stack
changed. Commit `pyproject.toml` and `uv.lock` together: `chore: upgrade <pkg> to <ver>` (or `fix:`
for a security patch, without detailing an unfixed vulnerability publicly).
