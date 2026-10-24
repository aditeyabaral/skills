# Autonomous workflow

How an agent takes a piece of work on pesu-auth from request to open pull request without a human
in the loop, and where it must stop. The `implement-feature` skill is the executable version of
this document; this file is the reference for the rules and the role handoffs.

## Boundaries

An agent working autonomously **may**: read anything in the repo; create branches; edit code, tests
and docs; run the app, the tests (live tests once, never alongside another live run), linters,
Docker builds and benchmarks against a local instance; commit; push **to the contributor's fork**; open a PR from the
fork to `pesu-dev/auth` `dev`; read CI results and push fixes to its own PR branch.

An agent **must not**: push to `pesu-dev/auth` or to any `dev`/`main` branch; merge, approve or
close PRs it did not open; trigger or call any deploy; change repository settings, secrets or
variables; commit `.env` or any credential; print credentials or personal data; weaken a test, a
coverage gate, a lint rule or a CI check to make something pass; force-push anything except its own
unmerged PR branch.

## Stop and hand back to a human when

- the change is **breaking** (see `.agents/pesudev-skills/auth/docs/api-contract.md`) and the
  request did not explicitly ask for a breaking change;
- the change touches security-sensitive behaviour in a way the request did not spell out (auth,
  logging of personal data, the metrics token), or a vulnerability is found (report privately);
- the work needs a new secret, environment variable, infrastructure, workflow permission, or a new
  runtime dependency the request did not mention;
- the request is ambiguous in a way that changes the design, and the codebase does not settle it;
- the same check still fails after **three** honest fix attempts;
- PESU Academy is down or behaving differently, so the change cannot be verified live;
- live credentials are needed and absent for a change to `app/pesu.py` or
  `app/models/upstream.py` (finish everything else and say the live verification is outstanding).

"Hand back" means: stop making changes, leave the branch in a clean, committed state, and write the
report described below with the exact question or blocker.

## The lifecycle

| #   | Phase        | Role (`.agents/pesudev-skills/auth/agents/`)             | Skills                                             | Exit criterion                                                                                                   |
| --- | ------------ | -------------------------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 1   | Triage       | `planner`                                                | `triage-issue`, `navigate-codebase`                | Problem restated, type classified, acceptance criteria written                                                   |
| 2   | Plan         | `planner`                                                | area skill (`add-endpoint`, `update-scraper`, ...) | Written plan: files to change, tests to add, docs to update, version impact, risks                               |
| 3   | Set up       | `implementer`                                            | `setup-environment`                                | Clean branch from `upstream/dev`, baseline tests green                                                           |
| 4   | Test first   | `test-engineer`                                          | `write-tests`                                      | New tests written and failing for the right reason                                                               |
| 5   | Implement    | `implementer` (+ `scraper-specialist` for upstream work) | area skill                                         | Tests pass; code follows `.agents/pesudev-skills/auth/docs/conventions.md`                                       |
| 6   | Verify       | `test-engineer`                                          | `run-tests`, `run-locally`                         | Full suite green with 100% line and branch coverage, lint clean, app and Docker smoke pass, live run if relevant |
| 7   | Document     | `docs-writer`                                            | `update-docs`                                      | README, `docs/`, OpenAPI docs, CONTRIBUTING and comments match behaviour                                         |
| 8   | Review       | `reviewer`, `security-auditor`                           | `review-change`, `security-review`                 | No blocking findings; every finding fixed or explicitly justified                                                |
| 9   | Release prep | `release-manager`                                        | `bump-version`, `open-pull-request`                | Version bumped once, commits clean, PR opened to `dev`                                                           |
| 10  | CI           | `ci-investigator`                                        | `fix-ci`                                           | All PR checks green, or a blocker reported                                                                       |

Loop back from 8 to 5 until the review is clean, and from 10 to 5 when CI finds something. Keep the
plan from phase 2 updated as reality changes it.

### Using roles

The files in `.agents/pesudev-skills/auth/agents/` describe roles. They are plain instructions, not
tied to any tool:

- If your tool can start a separate sub-agent, start one per role and give it the role file's
  contents plus the inputs the role lists. Run the reviewer and security auditor as separate
  sub-agents from the implementer: a fresh context reviews more honestly.
- If it cannot, perform the role yourself: read the role file, follow it for that phase, and write
  its output before moving to the next phase.
- In GitHub Copilot, every role is also a custom agent: pick it in the agent picker, or assign an
  issue to it. The custom agent (`.github/agents/<role>.agent.md`) is a generated wrapper that
  tells Copilot to read the role file, so the role file is still what it follows.

### Copilot's cloud agent

Copilot's cloud agent (an issue assigned to Copilot) pushes a branch to the repository it runs in
and opens the pull request there itself. Pull requests to `pesu-dev/auth` come from forks
(`.github/CONTRIBUTING.md`), so run it on your fork instead: sync the fork's `dev` so it has the
custom agents, assign the issue there, and once Copilot's pull request inside the fork is ready,
open the fork → `pesu-dev/auth` `dev` pull request yourself. The release-manager phase then only
checks the version bump and the pull request description.

The cloud agent also runs behind GitHub's firewall by default, which blocks pesuacademy.com and the
Render deployments unless a maintainer allowlists them. Tests that call PESU Academy fail there for
that reason alone. Run `uv run pytest tests/unit -q`, report the run as partial, and leave the full
`scripts/run_tests.py` run to CI and to a human with credentials.

## Definition of done

- [ ] Acceptance criteria from triage are met and demonstrated (test names or command output).
- [ ] `uv run python scripts/run_tests.py` passes, with 100% line and branch coverage.
- [ ] Live tests run once if the change touches `app/pesu.py`, `app/models/upstream.py` or
  `/authenticate` and credentials exist; otherwise the PR says they were not run.
- [ ] `uv run pre-commit run --all-files` passes.
- [ ] Docker image builds, runs as UID 10001 and answers on `/health` when the change affects
  runtime, dependencies or the `Dockerfile`.
- [ ] README, `docs/`, OpenAPI docs (`app/docs/`) and comments updated for any behaviour change.
- [ ] Review and security review have no open blocking findings.
- [ ] Version bumped exactly one step (minor by default, patch or major where they apply), `uv.lock` in step.
- [ ] Commits follow Conventional Commits with explanatory bodies.
- [ ] PR opened from the fork to `dev` with the template filled in truthfully.

## Reports

Every role ends with a short written report; the final report to the human is the last one. Use
this shape:

```markdown
## Outcome
<done | blocked | handed back>: one sentence.

## What changed
- <file>: <what and why>

## Verification
- <command> → <result, with counts> (say "partial" if live tests were skipped)

## Open items
- <anything not done, not verified, or needing a decision>
```

Never claim a check passed that was not run, and never omit a failure.
