---
name: triage-issue
description: Turn a pesu-auth GitHub issue or a loosely worded request into a classified, reproducible problem with acceptance criteria and the right labels and next skill. Use at the start of work that begins from an issue, a bug report, a feature request or a user complaint.
---

# Triaging work for pesu-auth

## 1. Read the source

```bash
gh issue view <n> --repo pesu-dev/auth --comments
gh issue list --repo pesu-dev/auth --search "<keywords>" --state all   # duplicates?
gh pr list --repo pesu-dev/auth --search "<keywords>" --state all      # already being fixed?
```

## 2. Classify

| Type                       | Signals                                                                                          | Next skill                                              |
| -------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| Bug                        | wrong status/body, 500, crash, leak                                                              | `fix-bug`                                               |
| Upstream change            | 422s, unexpected 502s, sudden 401s for valid users, nulls where values were, parse-error metrics | `update-scraper` (via `diagnose-production` if in prod) |
| Feature: new route         | "add an endpoint"                                                                                | `add-endpoint`                                          |
| Feature: fields/validation | new request option, new response/profile field                                                   | `change-api-models`, `update-scraper`                   |
| Observability              | "we can't see X"                                                                                 | `add-metric`                                            |
| Dependencies/tooling       | upgrade, Python version, pre-commit                                                              | `update-dependencies`                                   |
| CI                         | a workflow fails or needs changing                                                               | `fix-ci`                                                |
| Docs                       | README/Swagger wrong or missing                                                                  | `update-docs`                                           |
| Refactor                   | no behaviour change intended                                                                     | `refactor`                                              |

Decide if it is **breaking** (`.agents/pesudev-skills/auth/docs/api-contract.md`). If it is and the request did not say so,
escalate before planning.

## 3. Reproduce (bugs)

Reproduce against a local instance (`run-locally`) or with a failing test. For production-only
symptoms, use `diagnose-production`. Record exact inputs (redact credentials) and the observed vs
expected output. If it does not reproduce, say so with what you tried.

## 4. Write acceptance criteria

Observable, testable statements, each of which will become a test:

```markdown
- [ ] POST /authenticate with {"fields": []} returns 400 with message containing "Fields must be a non-empty list"
- [ ] /metrics?fmt=json includes "fooTotal", counted once per ...
```

## 5. Labels (suggest, do not apply unless you are a maintainer)

From `.github/CONTRIBUTING.md`: `bug`, `enhancement`, `feature`, `documentation`, `tests and ci/cd`,
`authentication`, `pesuacademy`, `student profile`, `api`, `good first issue`, `help wanted`.

## Output

Problem statement, type, breaking yes/no, reproduction (or why not), acceptance criteria, suggested
labels, and the next skill to load.
