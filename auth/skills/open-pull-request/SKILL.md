---
name: open-pull-request
description: Turn a finished pesu-auth branch into a pull request - Conventional Commits with explanatory bodies, rebase on upstream/dev, final checks, push to the contributor's fork, and gh pr create to pesu-dev/auth dev with the PR template filled in truthfully. Use as the last step of any change. Never merges.
---

# Opening a pull request

Policy (enforced by `source.yaml`): the PR comes **from a fork**, from a branch that is **not
`main`**, and targets **`dev`** of `pesu-dev/auth`.

## 1. Preconditions

- Definition of done in `.agents/pesudev-skills/auth/docs/autonomous-workflow.md` is met.
- Version bumped once (`bump-version`).
- `git status` is clean; `.env`, benchmark results and scratch files are not staged (`.gitignore`
  covers `.env`, `*.csv`, `*.png`).

## 2. Commits

Conventional Commits, matching `git log` in the repo:

```
<type>[!]: <what changed, imperative, lowercase, no period>

<Why the change was needed: the problem, who it affects.>

<How it works and why this approach, including alternatives rejected and library behaviour relied
on. Mention anything a reviewer would otherwise have to rediscover.>

<Verification: tests added, counts, coverage, live tests run or not.>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`; `!` after the type (and a
`BREAKING CHANGE:` paragraph) for incompatible changes. Keep the version bump in its own
`chore: bump version to X.Y.Z` commit. Wrap bodies at about 100 columns. Add any attribution
trailers your tool or the human requires.

Tidy history before pushing if it contains fixup noise:
`git rebase -i` is interactive; instead use `git reset --soft upstream/dev` and recommit in logical
units, or leave the history as is (PRs are squash-merged).

## 3. Rebase and final checks

```bash
git fetch upstream
git rebase upstream/dev
uv sync --all-groups
uv run pre-commit run --all-files
```

Re-check the version against the new base (`bump-version`) if `dev` moved.

## 4. Push to the fork

```bash
git remote -v                                # origin must be YOUR fork, not pesu-dev/auth
git push -u origin HEAD
```

If `origin` is `pesu-dev/auth`, stop: you need a fork (`gh repo fork pesu-dev/auth --remote`),
then push there. Never push to `pesu-dev/auth`.

## 5. Create the PR

Title: the Conventional Commit line of the change (e.g. `feat: add apiVersion to /health`). The
template's `#IssueNumber - ...` title comment is outdated (`.agents/pesudev-skills/auth/docs/known-issues.md`); put the issue in
the body instead.

Body: `.github/PULL_REQUEST_TEMPLATE.md`, filled in truthfully:

- **Description:** purpose, problem, context; `Fixes: #<n>` / `Related: #<n>`; delete the HTML comment.
- **Type of Change:** tick only what applies (breaking changes must be ticked and explained).
- **How Has This Been Tested?:** tick the suites you actually ran; test configuration (OS,
  `Python 3.14 via uv`, Docker build tested or not). State plainly if live tests were not run.
- **Checklist:** tick only items actually done; leave others unticked.
- **Affected API Behaviour:** tick the files touched.
- **Screenshots / API Demos:** real `curl` output for behaviour changes (redact credentials and
  personal data). Mandatory for breaking changes.
- **Additional Notes:** limitations, follow-ups, anything not verified.

```bash
gh pr create --repo pesu-dev/auth --base dev \
  --head <fork-owner>:<branch> \
  --title "feat: ..." --body-file /path/outside/repo/pr-body.md
```

End the body with any attribution line your tool or the human requires.

## 6. After opening

Report the PR URL. Watch the checks (`gh pr checks <n> --repo pesu-dev/auth --watch`) and load
`fix-ci` for any failure. Fork PRs never run the live tests in CI. Do not merge, request merges, or
ping maintainers repeatedly.
