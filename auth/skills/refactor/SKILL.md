---
name: refactor
description: Restructure pesu-auth code without changing behaviour - extract or move functions, simplify, rename, remove duplication - while proving equivalence with the existing tests, OpenAPI schema and metrics output. Use when the goal is code quality rather than new behaviour.
---

# Refactoring safely

A refactor changes **no** observable behaviour: not a status code, a message, a JSON key, a log
level, a metric name or label, a timing guarantee, or a close/cancel path.

## Before

```bash
uv run python scripts/run_tests.py > /tmp/before-tests.txt 2>&1
uv run python -c "import json; from app.app import app; print(json.dumps(app.openapi(), sort_keys=True))" > /tmp/before-openapi.json
```

Coverage must be 100% on the code you are about to move (`--cov-report=term-missing`). If it is
not, add characterisation tests first, in a separate commit.

## During

- Small steps, each leaving the suite green. Commit each step (`refactor: ...`).
- Keep public names that tests or docs refer to, or update every reference (`rg -n "<name>"`).
- Keep the patch points tests rely on: `app.app.metrics`, `app.app.pesu_academy`,
  `app.metrics.auth.METRICS_TOKEN`, `app.pesu.httpx2.AsyncClient` (the `wire` fixture) and
  `app.pesu.httpx2.AsyncClient.post` (the `upstream` fixture) are patched by name.
  Moving them breaks tests silently or loudly; update the tests in the same commit if you must.
- Preserve every why-comment that still applies; move it with the code.
- Do not mix in behaviour changes or dependency upgrades. If you find a bug, note it and fix it in a
  separate `fix:` change.

## After

```bash
uv run python scripts/run_tests.py
uv run python -c "import json; from app.app import app; print(json.dumps(app.openapi(), sort_keys=True))" > /tmp/after-openapi.json
diff /tmp/before-openapi.json /tmp/after-openapi.json   # must be empty
uv run pre-commit run --all-files
```

Also compare `/metrics` output for the same requests before and after if the refactor touches
middleware, handlers or `app/pesu.py`. For `app/pesu.py` concurrency code, follow
`change-upstream-client` as well.

Version: a refactor still needs the per-PR version bump (`bump-version`).
