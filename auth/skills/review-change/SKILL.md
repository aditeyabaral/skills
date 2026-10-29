---
name: review-change
description: Review a pesu-auth diff (your own before a PR, or someone else's PR) against a complete checklist - correctness, API contract, error semantics, upstream client safety, metrics rules, tests, docs, conventions, versioning - verifying claims by running code, and report findings with failure scenarios. Use before opening any PR and when asked to review a PR.
---

# Reviewing a change

## Get the diff

```bash
git fetch upstream
git diff upstream/dev...HEAD --stat
git diff upstream/dev...HEAD
git status                                   # uncommitted work too
# someone else's PR:
gh pr view <n> --repo pesu-dev/auth
gh pr diff <n> --repo pesu-dev/auth
gh pr checkout <n> --repo pesu-dev/auth      # to run it
```

Read each changed file **in full** around the hunks. Read the linked issue or plan so you can judge
completeness and scope.

## Checklist

**Correctness**

- Does it do what the acceptance criteria say, for every input class (valid, invalid, empty,
  boundary, wrong type, unknown key)?
- Every exit path of each changed function: success, each exception, cancellation.
- Async: no blocking calls on the event loop (parsing in `to_thread`), no lock held across an await
  on upstream, background tasks strong-referenced and their exceptions retrieved.

**API contract** (`.agents/pesudev-skills/auth/docs/api-contract.md`)

- camelCase on the wire, strict models, `extra="forbid"` on requests.
- Status codes follow the table; errors raised as `PESUAcademyError` subclasses; standard error body.
- Is anything backwards-incompatible? If yes: major bump, `!` commit, called out in the PR.

**Upstream client** (`change-upstream-client` invariants), if `app/pesu.py` or `app/models/upstream.py` changed

- Clients closed exactly once on every path; one client per login; `_upstream_call` around every
  upstream request; transport errors mapped to the step's error.
- Values returned as PESU sends them, from one source, or null; a changed field null and counted, not
  a failed login; no new field read that the API does not return; parse errors never logged or
  chained.

**Metrics** (`.agents/pesudev-skills/auth/docs/metrics.md`), if recording changed

- Bounded labels; one fact in one family; registry, JSON model, both doc examples and the
  `docs/metrics.md` row all updated; invariants still hold.

**Security** (`security-review`)

- No passwords, tokens or new personal data in logs, messages, labels or responses.

**Tests** (`.agents/pesudev-skills/auth/docs/testing.md`)

- New behaviour and each new branch tested; regression test for a bug; assertions meaningful (not
  inside a `pytest.raises` block, not on a mock); fresh collectors; no network in unit tests; 100%
  line and branch coverage.
- Test constants in a `conftest.py`, reached through fixtures; no section marker comments.

**Docs**

- README, `docs/`, `app/docs/`, docstrings, comments and CONTRIBUTING describe the new behaviour;
  comments explain why, in a line or two; stale comments updated.
- Errors raised with their fixed `message`; specifics in `detail`.

**Conventions and hygiene**

- Google docstrings with typed Args/Returns, annotations everywhere in `app/`, `TYPE_CHECKING`
  imports not used in FastAPI signatures, no debug prints, no commented-out code, no unrelated
  changes.

**Release**

- `version` raised by exactly one step versus `upstream/dev` (the right one of minor, patch or
  major), `uv.lock` matches, Conventional Commit
  messages with real bodies.

## Verify, do not assume

Run the targeted tests and the full suite (`run-tests`). For each suspicion, try to break it: a
`TestClient` request with the edge case, a unit test with a cancellation, a `curl` against
`run-locally`. A finding is only blocking if you can state the failure scenario.

## Report

```markdown
## Verdict
approve | changes required

## Findings (most severe first)
1. [blocking|should-fix|nit] `app/pesu.py:212`: <problem>. Failure scenario: <inputs → wrong result>. Fix: <suggestion>.

## Checked and fine
- <area>: <how verified>
```

When reviewing someone else's PR for a maintainer, also follow `review-pull-request`: fetching it
safely, posting suggestions, re-review rounds and what a merge needs. Post only if the human asked
you to; never approve or merge.
