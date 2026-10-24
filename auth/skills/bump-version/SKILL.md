---
name: bump-version
description: Bump the pesu-auth project version by exactly one step once per pull request - minor by default, patch for work meant for the last minor, major for breaking changes - update uv.lock to match, and verify with the same script the Version check job runs. Use when preparing any PR to pesu-dev/auth.
---

# Bumping the version

The *Version check* job of CI Checks fails a PR unless `project.version` in `pyproject.toml` is
**exactly one step** above the base branch's **and** `uv.lock` records the same version. The version
is the API version shown in `/openapi.json`, and the production release tags it `vX.Y.Z`.

## Rules (from `.github/scripts/check_version_bump.py` and CONTRIBUTING)

- **One bump per PR**, relative to `upstream/dev`, not per commit or feature.
- **Minor** (`x.Y+1.0`): the default, whatever the change: features, fixes, docs, refactors,
  dependencies.
- **Patch** (`x.y.Z+1`): anything (a feature, fix, docs or tests) that was meant to be part of the
  last minor release but was missed or split out into its own pull request. It does not mean "a
  small fix".
- **Major** (`X+1.0.0`): backwards-incompatible API or schema changes
  (`.agents/pesudev-skills/auth/docs/api-contract.md`).
- Anything else fails: an unchanged version, a skipped one, or one that does not reset the numbers to
  its right (`4.19.1` from `4.18.0`). If the right step is unclear, propose it to the human with the
  reason rather than guessing.

## Steps

```bash
git fetch upstream
git show upstream/dev:pyproject.toml | grep -m1 '^version'   # base version, e.g. version = "5.6.0"
grep -m1 '^version' pyproject.toml                            # yours
```

If your branch already bumps one step above the base (for example after a rebase where `dev` did not
move), do nothing. If `dev` moved, bump again from the new base.

Edit `version = "X.Y.Z"` in the `[project]` table of `pyproject.toml`, then:

```bash
uv lock                                       # records the new version for pesu-auth in uv.lock
python3 .github/scripts/check_version_bump.py \
  --base-pyproject <(git show upstream/dev:pyproject.toml) \
  --head-pyproject pyproject.toml --head-lock uv.lock --base-ref dev
git add pyproject.toml uv.lock
git commit -m "chore: bump version to X.Y.Z"
```

The check must end with `✅ Version raised from ... to ... (a <minor|patch|major> bump), with uv.lock in step.`

`uv lock` should change only the `pesu-auth` version entry. If it rewrote other packages, you ran it
with an upgrade flag or a changed dependency; inspect `git diff uv.lock` and keep dependency changes
in their own commit with their own justification. If it rewrote the `revision` line, your uv is older
than CI's: `uv self update`, then relock. Never edit `uv.lock` by hand.

Dependabot's submodule bumps skip the check; a PR from a fork that supersedes one does not.
