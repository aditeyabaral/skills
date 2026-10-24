# Code conventions

What code in pesu-auth looks like. Match it; reviewers will ask for it and CI enforces most of it.

## Language and tooling

- Python **3.14+** (`requires-python = ">=3.14"`, ruff `target-version = "py314"`). Use modern syntax:
  `X | None`, `match`, walrus (`:=`) where it reads well, `StrEnum`, PEP 695 generics if useful.
- uv manages everything. `uv sync --all-groups` for development; run tools with `uv run ...`.
  Never `pip install` into the environment.
- Line length **120**, double quotes, formatted by `ruff format`.

## Ruff rules (`pyproject.toml`)

Selected: `E`, `F`, `W`, `UP`, `I`, `C90`, `D` (Google convention), `ANN`, `TYP`, `N`, `Q000`,
`RET`, over the whole repository except `AGENTS.md` and `.agents/`. Under `tests/`, `ANN` and `D1`
are ignored: tests need no annotations or docstrings, but every other rule applies. What this means
in practice:

- **Every** module, class, function and method in `app/` and `scripts/` has a docstring, Google
  style, with `Args:`, `Returns:`, `Raises:`, `Yields:` sections where they apply. Existing code
  writes the type in the section: `name (type): description.`
- **Every** parameter and return is annotated, including `-> None`.
- Imports sorted by ruff (`I`). Type-only imports go under `if TYPE_CHECKING:` with
  `from __future__ import annotations` at the top of the module.
- Keep functions small enough for `C90` (McCabe complexity).

Pre-commit runs ruff from `uv.lock` (`uv run --frozen ruff ...`), so it is the same version locally
and in CI. Run `uv run ruff check . && uv run ruff format --check .`, or let pre-commit fix things:
`uv run pre-commit run --all-files`.

## The `TYPE_CHECKING` trap with FastAPI

FastAPI resolves dependency and route annotations at runtime with `get_type_hints()`. A name that
only exists under `TYPE_CHECKING` in a module using `from __future__ import annotations` becomes a
`NameError` when the route is built. So **names used in FastAPI signatures (dependencies, `Security`,
request/response types) must be imported at runtime**. `app/metrics/auth.py` has the comment
explaining this for `HTTPAuthorizationCredentials`. In `app/app.py`, `Request` and `Response` are
under `TYPE_CHECKING` only because they are used in middleware/handler signatures FastAPI does not
introspect; do not copy that pattern into a route or dependency.

Pydantic models are evaluated at runtime too, which is why ruff is told
`runtime-evaluated-base-classes = ["pydantic.BaseModel"]`.

## Comments

The codebase comments **why**, densely, and almost never **what**. A comment explains a constraint,
a failure it prevents, or why the obvious alternative is wrong, often citing the incident or
library behaviour behind it. Examples worth imitating: the `_close_client_quietly` docstring and the
`cancelled` comment in `_upstream_call` (`app/pesu.py`), the `compare_digest` byte-encoding comment
in `app/metrics/auth.py`, the `finally` comment in `record_request_metrics`.

- Write that kind of comment for every non-obvious decision you make, and keep a new one to a line or
  two: the why only, not an essay on the alternatives. Maintainers ask for long ones to be trimmed.
- Do not narrate code (`# increment the counter`), and remove a comment that adds nothing.
- When you change behaviour a comment describes, update the comment in the same change.

## Logging

- Use the module-level `logging` functions (`logging.info(...)`), as the code does. f-strings are
  the house style for messages.
- **Never log a password, a token, a request body, a raw validation error's `input`, or a pydantic
  `ValidationError` from parsing PESU's responses.** The validation handler strips `input` because
  for a missing field it is the whole body; upstream parse failures log
  `_validation_failure_summary(e)` and raise `from None` because the error quotes personal data.
- Level follows severity: DEBUG for step-by-step tracing, INFO for lifecycle and outcomes, WARNING
  for expected failures (4xx, recoverable upstream issues), `logging.exception` only for faults that
  deserve a traceback (5xx, unexpected exceptions).
- The username (`user=...`) and the profile are logged at INFO by design, for tracing a login; do
  not remove them. Do not add logging of tokens or of personal data beyond those; see
  `.agents/pesudev-skills/auth/docs/security.md`.

## API style

- JSON keys are **camelCase** on the wire. Models use `alias_generator=to_camel`, and responses are
  dumped with `by_alias=True`. Python attribute names stay snake_case.
- Models are `strict=True`. Request models are `extra="forbid"` so unknown or deprecated keys are a
  400, not silently ignored.
- Every error body is `{status: false, message, timestamp}`. Raise a `PESUAcademyError` subclass to
  produce one; never build error `JSONResponse`s in routes. Leave `message` as the class's fixed
  default and put the specifics (the username, what PESU answered) in `detail=`, which only the log
  sees.
- A profile value PESU has not got is `null`, never absent and never a placeholder; values are
  returned as PESU sends them (`.agents/pesudev-skills/auth/docs/upstream.md`).
- Every route has `tags=[...]`, `responses=<docs>.response_examples`, a docstring (it becomes the
  OpenAPI description) and a docs module in `app/docs/`. See `.agents/pesudev-skills/auth/docs/api-contract.md`.

## Naming

- Modules and functions snake_case, classes PascalCase, constants UPPER_SNAKE.
- Exceptions end in `Error` and live in `app/exceptions/<area>.py`.
- Metric family constants are UPPER_SNAKE in `app/metrics/collector.py`; exposed names are
  `pesu_auth_<name>` with `_total` for counters and `_seconds` for durations.
- Test files and functions start with `test_` (enforced by the `name-tests-test` hook). Test
  constants live in a `conftest.py` and reach tests through fixtures; see
  `.agents/pesudev-skills/auth/docs/testing.md`.
- In anything a caller reads (the README, model descriptions, Swagger examples), do not mention "the
  mobile API": to a caller there is only PESU Academy. And nowhere, code and tests included, call a
  profile field "new" or "core": there is one field list, and every field behaves the same.

## Commits

Conventional Commits: `feat:`, `fix:`, `docs:`, `style:`, `refactor:`, `test:`, `chore:`; a `!` or a
`BREAKING CHANGE:` footer for incompatible changes. The body explains **why** and what a reviewer
needs to know (see `git log` for the house style: several paragraphs for a real change is normal).
The version bump is its own commit: `chore: bump version to X.Y.Z`.

## Markdown

`mdformat` with GFM runs on every `.md` in pre-commit. Tables are realigned and lists renumbered to
`1.`; do not fight it. Keep tables aligned by running the hook rather than by hand. Its
`mdformat-frontmatter` plugin keeps YAML frontmatter intact; without it the frontmatter of the
Copilot custom agents in `.github/agents/` is rewritten as Markdown and Copilot stops seeing them.
