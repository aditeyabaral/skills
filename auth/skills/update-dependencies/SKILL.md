---
name: update-dependencies
description: Add, upgrade or remove pesu-auth dependencies, dev tools, pre-commit hooks, the Python version or base images with uv, keeping uv.lock, pyproject.toml, the Dockerfile, CI and pre-commit pins consistent and verified. Use for dependency bumps, security patches, new packages or toolchain upgrades.
---

# Updating dependencies

Runtime deps are in `[project].dependencies`, dev tools in `[dependency-groups].dev`
(`pyproject.toml`), all locked in `uv.lock`. The Docker image installs from the lock with
`uv sync --no-dev --frozen`, so the lock is what production runs.

## Adding a package

Justify it first: can the standard library or an existing dependency do it? New runtime
dependencies need maintainer agreement (escalate unless requested).

```bash
uv add "<package>>=<version>"            # runtime
uv add --group dev "<package>>=<version>"  # dev only
```

## Upgrading

```bash
uv lock --upgrade-package <package>      # one package
uv lock --upgrade                        # everything (separate PR, larger review)
uv sync --all-groups
```

Raise the lower bound in `pyproject.toml` when the code starts relying on the new version (the
project pins `>=` floors, and past bumps raised them together, e.g. `fastapi>=0.141.1`). Read the
changelogs of `fastapi`, `starlette`, `pydantic`, `httpx2` and `selectolax` for behaviour changes;
note anything relevant in the PR.

Known sensitive points when upgrading:

- **FastAPI/Starlette:** the OpenAPI override (`_openapi_without_phantom_validation_errors`), exception
  middleware ordering, `HTTPBearer(auto_error=False)` behaviour, `TestClient`. The OpenAPI and
  metrics tests catch most regressions.
- **pydantic:** strict mode and alias behaviour, error message text (the 400 message is built from
  it and some tests assert it).
- **httpx2:** client lifecycle and exception types.
- **ruff:** the pre-commit hook `rev` in `.pre-commit-config.yaml` and the `ruff` dev dependency
  should move together, or local and hook results diverge. Run `uv run ruff check .` and fix new
  findings in the same PR.

## Pre-commit hooks

```bash
uv run pre-commit autoupdate
uv run pre-commit run --all-files
```

## Python version

The floor is in `requires-python`, ruff's `target-version`, the `Dockerfile` base images
(`ghcr.io/astral-sh/uv:python3.14-alpine`, `python:3.14-alpine`), `actions/setup-python`
`python-version` in `.github/workflows/lint.yaml`, `pre-commit.yaml` and
`copilot-setup-steps.yml`, README and CONTRIBUTING prerequisites. Change all of them together. Raising the floor is a breaking change for anyone
running from source: major version.

## requirements.txt

It is not used by the build and is stale (`.agents/pesudev-skills/auth/docs/known-issues.md`). If you touch dependencies,
regenerate it (`uv pip compile pyproject.toml -o requirements.txt`) so it at least matches, or
propose its removal in the PR; do not hand-edit it.

## Verify

```bash
uv run python scripts/run_tests.py
uv run pre-commit run --all-files
docker build . --tag pesu-auth                  # the image must build from the lock
uvx --python 3.14 pip-audit --strict --require-hashes --disable-pip -r <(uv export --frozen --no-dev --no-emit-project)
```

Then the Docker smoke test from `run-locally`, and one live run if `httpx2`, `selectolax` or the
HTTP stack changed. Commit `pyproject.toml` and `uv.lock` together: `chore: upgrade <pkg> to <ver>`
(or `fix:` for a security patch, without detailing an unfixed vulnerability publicly).
