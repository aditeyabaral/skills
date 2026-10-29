---
name: implement-feature
description: Take any pesu-auth change from request to an open pull request autonomously - triage, plan, branch, test-first implementation, verification, docs, independent review, version bump, PR and CI - coordinating the role files and the area skills. Use for every feature, fix or change unless the task is explicitly a single narrow step.
---

# Implementing a change end to end

This is the orchestrator. It calls the other skills and the roles in
`.agents/pesudev-skills/auth/agents/`. The rules, boundaries and definition of done are in
`.agents/pesudev-skills/auth/docs/autonomous-workflow.md`; read it once before starting.

**Roles:** if your tool can start sub-agents, run each role below as its own sub-agent, passing the
role file and the inputs it lists. Otherwise follow the role file yourself for that phase. Run the
reviewer and security auditor with a fresh view of the diff, not from memory of writing it.

Keep a running work log (plan, decisions, commands and results); the final report is built from it.

## Phase 1: Triage and plan (role: `planner`)

1. Load `triage-issue`, then `navigate-codebase`.
1. Produce the planner's output: problem, acceptance criteria, file-level plan, tests, docs, version
   impact, risks.
1. **Gate:** if the change is breaking and not explicitly requested, needs secrets, infrastructure or
   an unrequested dependency, or the request is ambiguous in a design-changing way, stop and hand
   back with the question.

## Phase 2: Branch and baseline (role: `implementer`)

```bash
git fetch upstream
git switch -c <type>/<short-slug> upstream/dev     # e.g. feat/health-version, fix/empty-fields-message
uv sync --all-groups
uv run pytest tests/unit -q                        # baseline must be green
```

If `upstream` is missing or the tree is dirty, load `setup-environment` first. Never work on `dev`
or `main`.

## Phase 3: Tests first (role: `test-engineer`)

Load `write-tests`. Write the tests for each acceptance criterion. Run them; they must fail for the
expected reason.

## Phase 4: Implement (role: `implementer`, plus `scraper-specialist` for PESU work)

Load the area skill(s) from the plan:

| Change                                         | Skill                    |
| ---------------------------------------------- | ------------------------ |
| new route                                      | `add-endpoint`           |
| the API Explorer page at `/`                   | `change-api-explorer`    |
| request/response/profile fields or validation  | `change-api-models`      |
| PESU's API, login, profile fields              | `update-scraper`         |
| new metric                                     | `add-metric`             |
| new failure mode                               | `add-error-type`         |
| environment variable or command-line flag      | `add-configuration`      |
| HTTP client, lifecycle, timeouts, cancellation | `change-upstream-client` |
| packages or tool versions                      | `update-dependencies`    |
| restructure only                               | `refactor`               |
| bug                                            | `fix-bug`                |

Work step by step, narrowest test first. Commit logical steps as you go (`open-pull-request` has
the commit format).

## Phase 5: Verify (role: `test-engineer`)

```bash
uv run python scripts/run_tests.py          # all green, 100% line and branch coverage
uv run pre-commit run --all-files           # lint, format, mdformat, hygiene, tests
```

Then, as relevant:

- live tests once, if `app/pesu.py`, `app/models/upstream.py` or `/authenticate` changed and `.env`
  has credentials:
  `uv run pytest -m secret_required -v`;
- `run-locally`: smoke-test affected endpoints on a running server, and the Docker image if runtime,
  dependencies or the `Dockerfile` changed.

## Phase 6: Documentation (role: `docs-writer`)

Load `update-docs`. README, `app/docs/`, CONTRIBUTING, `.env.example`, docstrings and comments must
describe the new behaviour. Re-run `uv run pytest tests/unit/test_openapi_docs.py -q`.

## Phase 7: Review (roles: `reviewer`, `security-auditor`)

Load `review-change` and `security-review` against `git diff upstream/dev...HEAD`. For every
blocking or should-fix finding: fix it (back to phase 4/5) or write down why it is not a problem.
Repeat until the review is clean. Three failed rounds on the same finding → hand back.

## Phase 8: Release prep (role: `release-manager`)

1. Walk the definition of done in `.agents/pesudev-skills/auth/docs/autonomous-workflow.md`; every box must be true.
1. Load `bump-version` (once per PR).
1. Load `open-pull-request`: rebase on `upstream/dev`, final `pre-commit`, push to the fork, open
   the PR to `dev`.

## Phase 9: CI (role: `ci-investigator`)

Load `fix-ci` if any check on the PR fails. Push fixes to the same branch. Remember that fork PRs do
not run the live tests in CI; state in the PR whether you ran them locally.

## Final report

Use the report format in `.agents/pesudev-skills/auth/docs/autonomous-workflow.md`: outcome, changes, verification with real
counts (say "partial" when live tests did not run), PR URL, open items. Never report a check you did
not run as passing.
