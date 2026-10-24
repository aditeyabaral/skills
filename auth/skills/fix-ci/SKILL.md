---
name: fix-ci
description: Diagnose and fix failing GitHub Actions checks on a pesu-auth PR or branch - Pre-Commit Checks, Lint, Docker Image Build, Enforce PR Origin Policy, Version Check - by reading the failed logs, reproducing locally and fixing the cause without weakening any check. Use whenever a check is red.
---

# Fixing CI

Workflow reference: `.agents/pesudev-skills/auth/docs/ci-cd.md`.

## 1. Find the failure

```bash
gh pr checks <n> --repo pesu-dev/auth
gh run list --repo pesu-dev/auth --branch <branch> --limit 5      # runs on the PR's head
gh run view <run-id> --repo pesu-dev/auth --log-failed | head -200
```

Read the first real error in the log, not the summary at the bottom. Redact anything sensitive
before quoting logs.

## 2. Diagnose by check

| Check                         | Typical causes                                                                                                                                                                                     | Reproduce and fix                                                                                                                                                                                                                                                                                                       |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pre-Commit Checks**         | ruff lint/format, mdformat rewrote a Markdown file, trailing whitespace/EOF, `name-tests-test`, `debug-statements` (a `breakpoint()`/`pdb`), test failures, coverage < 95%                         | `uv run pre-commit run --all-files`; commit what the fixers changed; fix failing tests; add tests for uncovered lines. The run summary says whether live tests were skipped (they are, on fork PRs)                                                                                                                     |
| **Pre-Commit: `sync-agents`** | `.github/agents/` no longer matches the roles in `.agents/pesudev-skills/auth/agents/`: usually a Dependabot submodule bump that changed a role's frontmatter, or someone edited a wrapper by hand | On your own PR: `uv run python scripts/sync_agents.py` and commit the regenerated `.github/agents/`. If it says the submodule is missing, run `git submodule update --init` first. On a Dependabot PR you cannot push to its branch: follow "Dependabot submodule bumps" in `.agents/pesudev-skills/auth/docs/ci-cd.md` |
| **Lint**                      | same ruff issues, stricter output                                                                                                                                                                  | `uv run ruff check . && uv run ruff format --check .`                                                                                                                                                                                                                                                                   |
| **Docker Image Build**        | image does not build (lock out of date, new file outside `app/` not copied), container does not answer on `/` within 5s (startup crash, e.g. an import error or the CSRF prefetch failing)         | the Docker steps in `run-locally`; `docker logs` shows the startup error                                                                                                                                                                                                                                                |
| **Enforce PR Origin Policy**  | PR from a branch in `pesu-dev/auth`, from a fork's `main`, or targeting `main`                                                                                                                     | Fix the PR: push the branch to your fork under a non-`main` name and open a new PR to `dev` (the base can be changed with `gh pr edit <n> --repo pesu-dev/auth --base dev`)                                                                                                                                             |
| **Version Check**             | version not raised vs the base, or `uv.lock` not updated                                                                                                                                           | `bump-version` (includes the exact local command)                                                                                                                                                                                                                                                                       |

Failures in `test_*_functional.py`/`test_app_integration.py` with connection errors or timeouts
usually mean PESU Academy was unreachable from the runner. Rerun once:
`gh run rerun <run-id> --repo pesu-dev/auth --failed` (needs permission on the repo; if you lack it,
push an empty change only if the human agrees, or report it).

## 3. Fix and push

Fix on the PR branch, commit with the right type (`style:`, `fix:`, `test:`, `chore:`), run
`uv run pre-commit run --all-files` locally, push to the fork. Checks re-run automatically.

## Never

Edit `.github/workflows/*`, `.pre-commit-config.yaml`, the coverage gate or ruff configuration to
make a failure go away; skip hooks with `--no-verify`; mark tests `skip`/`xfail` to get green. If a
check itself is wrong, say so in the report and leave the decision to a maintainer.

Three attempts on the same failure without progress → stop and report with the log excerpt.
