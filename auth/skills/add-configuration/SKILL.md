---
name: add-configuration
description: Add or change a pesu-auth setting - an environment variable the app reads or a command-line flag - deciding when it is read, validating it so a bad value stops the app at startup, testing it, documenting it in docs/development.md (which test_docs.py enforces), README and .env.example, and handing the values each Render service needs to a maintainer. Use whenever the app should read something new from its environment or command line, or an existing setting changes.
---

# Adding a setting

Settings the app has today:

| Setting                 | How it is read                                                                                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--host`, `--debug`     | `_build_arg_parser()` in `app/app.py`                                                                                                                              |
| `--port`, `PORT`        | `--port`, whose default comes from `PORT` (Render sets it)                                                                                                         |
| `METRICS_TOKEN`         | Once at import, in `app/metrics/auth.py`: the dependency that checks it is built then. Blank means unset                                                           |
| `PESU_AUTH_ENVIRONMENT` | Once at startup, in the lifespan: checked against `Environment` (`app/models/health.py`), kept on `app.state.environment`, reported by `/health`. Blank is invalid |

The app never reads `.env`; only the tests and the benchmark scripts do.

**Working autonomously, a new environment variable is a reason to stop and hand back**
(`.agents/pesudev-skills/auth/docs/autonomous-workflow.md`), unless the request asked for it: a
maintainer has to agree to it and set it on Render.

## 1. Decide

1. **Does it need to be a setting?** Only if deployments or developers need different values; a
   constant in the code is simpler and needs no Render change.
1. **When is it read?** Once at startup in the lifespan, onto `app.state`, unless something needs it
   while modules are imported (a dependency built at import, like `METRICS_TOKEN`). Never per
   request: a setting must not change halfway through a process's life.
1. **Read it with `os.environ.get("NAME", default)` or `os.getenv("NAME")`, written out literally.**
   `tests/unit/test_docs.py` finds the variables the app reads with that pattern and fails if
   `docs/development.md` leaves one out; `os.environ["NAME"]` or a computed name slips past it.
1. **The default** is the right value for a local run and for CI's container smoke test, which
   starts the image with nothing set.
1. **Unset and blank.** Say which a blank value means: unset (`METRICS_TOKEN`, so a declared but
   empty variable turns the feature off) or invalid (`PESU_AUTH_ENVIRONMENT`).
1. **Validate, and fail at startup.** A setting with a fixed set of values gets a `Literal` alias
   checked with `typing.get_args`, as `Environment` is. A bad value raises `ValueError` in the
   lifespan, naming the variable, the allowed values and the value it got: uvicorn then exits with
   code 3, a Render deploy fails instead of going live, and Deploy to Production rolls back. Never
   fall back to the default for a value that is set but wrong.
1. **Secrets** (a token, a key): never logged or echoed in an error, compared with
   `secrets.compare_digest`, and given a placeholder, never a real value, in `.env.example`. Run
   `security-review`.
1. **Name** a new variable `PESU_AUTH_<NAME>`, as `PESU_AUTH_ENVIRONMENT` is, so it cannot clash with
   one the platform sets.

A command-line flag goes in `_build_arg_parser()` with help text; if it has an environment
variable, read that as the flag's default, as `--port` does with `PORT`.

## 2. Test it

Follow `write-tests`. Values go in `tests/unit/conftest.py` and reach the tests through
parametrized fixtures (`environment` and `invalid_environment` are the model).

- **Read at startup:** set it with `monkeypatch.setenv` (or remove it with `monkeypatch.delenv`),
  then enter `with TestClient(app) as client:`, which runs the lifespan.
- **Invalid values:**
  `with pytest.raises(ValueError, match="..."):` around `with TestClient(app): pass`.
- **Read at import:** the value is fixed before any test runs, so patch the module attribute
  (`monkeypatch.setattr("app.metrics.auth.METRICS_TOKEN", ...)`), and add an autouse fixture in
  `tests/conftest.py` that pins it for every other test, as `_metrics_token_unset` does. Otherwise a
  value in a developer's `.env` (which `load_dotenv()` loads) or shell breaks unrelated tests.
- **A flag:** build the parser and parse arguments, as the `PORT` tests in
  `tests/unit/test_app_unit.py` do.

## 3. Document it

- A row in the Configuration table in `docs/development.md`: the setting, its default and what it
  does, including what happens with a bad value.
- The README, where callers see the effect (as the `/health` section explains `environment`).
- `.env.example`: a commented-out line with a comment saying the app reads it from its environment,
  not from `.env`, which only the tests and benchmarks read.
- `_build_arg_parser()` help text for a flag.
- In pesu-dev/skills: the Configuration section of `.agents/pesudev-skills/auth/docs/architecture.md`,
  and a row in `setup-environment`'s troubleshooting table if a value exported in the shell can break
  the tests.

## 4. Hand the deployment to a maintainer

Agents never change Render. In the pull request description, list for each service the exact value
to set:

- pesuauth-dev (staging): **before the merge**, because a merge to `dev` deploys staging at once;
- pesuauth (production): before the next release.

A secret's value never goes in the PR; say only that it must be set.

## 5. Verify

- `uv run pytest tests/unit -q`, then `uv run python scripts/run_tests.py`.
- Run the app with the setting unset, with a good value and with a bad one:
  `PESU_AUTH_<NAME>=... uv run python -m app.app`. A bad value must stop it with your message and a
  non-zero exit code.
- The Docker image starts with nothing set (`run-locally`), as CI's smoke test will check.
