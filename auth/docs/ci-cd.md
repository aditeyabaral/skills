# CI/CD and branching

## Branches and remotes

- `pesu-dev/auth` has two long-lived branches: **`dev`** (integration, deployed to staging) and
  **`main`** (production, only ever fast-forwarded from `dev` by the prod workflow). The default
  branch is `dev`.
- Contributors work on a **fork**. In a typical clone, `origin` is the fork and `upstream` is
  `https://github.com/pesu-dev/auth`. Check with `git remote -v`; add upstream if missing:
  `git remote add upstream https://github.com/pesu-dev/auth`.
- Feature branches are named for the change (`feat/metrics-bearer-token`, `fix/client-leak-...`,
  `docs/...`), created from an up-to-date `upstream/dev`.
- Nobody pushes to `dev` or `main` directly; changes land by PR into `dev`, squash-merged with a
  Conventional Commit title and `(#<PR number>)`.

## Workflows (`.github/workflows/`)

| Workflow                                        | Trigger                                                                | What it checks                                                                                                                                                                                                                                                                                                                                                             | Reproduce locally                                                                                                                                                           |
| ----------------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pre-commit.yaml` (Pre-Commit Checks)           | every push and PR                                                      | checkout with the `.agents/pesudev-skills` submodule, `uv sync --all-groups`, then `uv run pre-commit run --all-files`: ruff, mdformat, file hygiene, test naming, debug statements, `sync-agents` (`.github/agents/` matches the roles), and the full test suite with the 95% gate. `TEST_*` secrets exist only for runs in the base repo (pushes to `dev`), not fork PRs | `uv run pre-commit run --all-files`                                                                                                                                         |
| `lint.yaml` (Lint)                              | pushes to non-`main`/`dev` branches, PRs                               | `ruff check . --output-format=github` and `ruff format . --check`                                                                                                                                                                                                                                                                                                          | `uv run ruff check . && uv run ruff format --check .`                                                                                                                       |
| `docker.yaml` (Docker Image Build)              | pushes to non-`main`/`dev` branches, PRs                               | builds the image, runs it on port 5000, `curl --fail http://localhost:5000`                                                                                                                                                                                                                                                                                                | see the `run-locally` skill                                                                                                                                                 |
| `source.yaml` (Enforce PR Origin Policy)        | PRs                                                                    | PR head is a **fork**, head branch is not `main`, base is **`dev`**                                                                                                                                                                                                                                                                                                        | check `gh pr view` fields                                                                                                                                                   |
| `version-check.yaml` (Version Check)            | PRs                                                                    | `.github/scripts/check_version_bump.py`: `project.version` strictly greater than the base's, and `uv.lock` records the same version                                                                                                                                                                                                                                        | `python3 .github/scripts/check_version_bump.py --base-pyproject <(git show upstream/dev:pyproject.toml) --head-pyproject pyproject.toml --head-lock uv.lock --base-ref dev` |
| `copilot-setup-steps.yml` (Copilot Setup Steps) | pushes and PRs that change it, manual dispatch                         | the environment GitHub Copilot's cloud agent starts in: checkout with the submodule, Python 3.14, `uv sync --all-groups`. No `TEST_*` secrets, so Copilot never runs live tests                                                                                                                                                                                            | none: it is only setup                                                                                                                                                      |
| `deploy-staging.yaml`                           | after Pre-Commit Checks succeeds on `dev`, or manual dispatch on `dev` | skips if `dev` moved past the validated commit; POSTs the Render staging deploy hook                                                                                                                                                                                                                                                                                       | none: maintainers only                                                                                                                                                      |
| `deploy-prod.yaml`                              | manual dispatch, users listed in `vars.PROD_DEPLOYMENT_ALLOWED_USERS`  | fast-forwards `main` to `dev`, pushes images `pesu-auth:<sha>` and `:latest` to Docker Hub and GHCR, deploys staging then production via Render hooks                                                                                                                                                                                                                      | none: maintainers only                                                                                                                                                      |

## Dependabot submodule bumps

`.github/dependabot.yml` opens a pull request against `dev` whenever pesu-dev/skills `main` moves,
bumping the `.agents/pesudev-skills` submodule. It is the one pull request allowed from a branch in
`pesu-dev/auth` itself (`source.yaml` waives the fork rule for `dependabot[bot]`), and the version
check is skipped for it.

If the bump adds or removes a role or changes a role's frontmatter, `sync-agents` fails on it,
because `.github/agents/` is out of date. Dependabot cannot fix that, and agents and contributors
cannot push to its branch. Either:

- a maintainer checks out the Dependabot branch, runs `uv run python scripts/sync_agents.py`, and
  pushes the regenerated `.github/agents/` to it; or
- anyone opens a pull request from their fork that bumps the submodule to the same commit
  (`git -C .agents/pesudev-skills checkout <sha>`, then `git add .agents/pesudev-skills`), runs the
  script, commits `.github/agents/`, and bumps the version as for any pull request. Dependabot
  closes its own pull request once `dev` already has that commit.

Agents take the second route. A bump that only changes role bodies, skills or docs passes as is.

## Deployment flow

1. PR merged into `dev` → Pre-Commit Checks on `dev` (with live tests) → staging deploy
   (`https://pesuauth-dev.onrender.com`).
1. A maintainer validates staging, then dispatches Deploy to Production → `main` fast-forwarded →
   images published → staging and production (`https://pesuauth.onrender.com`) deployed.
1. The deployed version is visible at `/openapi.json` → `info.version` and on the README badges.

Rollback: there is no automated rollback. The options are a revert PR into `dev` followed by a
normal release, or a maintainer redeploying a previous image in Render. Agents never perform either.

## Things agents must never do

- Push to `upstream` (pesu-dev/auth) or to any `dev`/`main` branch; merge PRs; approve PRs.
- Dispatch `deploy-staging.yaml` or `deploy-prod.yaml`, or call Render deploy hooks.
- Edit workflow secrets, repository variables or `PROD_DEPLOYMENT_ALLOWED_USERS`.
- Change workflows to skip or weaken a check to get a PR green.

## Reading CI results

```bash
gh pr checks <pr-number> --repo pesu-dev/auth
gh run list --repo pesu-dev/auth --branch <branch> --limit 5
gh run view <run-id> --repo pesu-dev/auth --log-failed
```

See the `fix-ci` skill for failure-by-failure diagnosis.
