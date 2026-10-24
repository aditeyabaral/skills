---
name: fix-bug
description: Fix a pesu-auth bug properly - reproduce it, capture it in a failing regression test, find the root cause, make the minimal fix, and check related paths and metrics. Use for wrong responses, 500s, crashes, leaks, wrong metrics or any behaviour that differs from the docs.
---

# Fixing a bug

## 1. Reproduce

- Get the exact input and the observed vs expected output (from `triage-issue`).
- Reproduce with the cheapest tool: a `TestClient` call with the upstream mocked (see the fixture
  recipes in `.agents/pesudev-skills/auth/docs/testing.md`), a unit call on `PESUAcademy`, or a running
  server (`run-locally`). Use the live account only when the bug depends on PESU's real HTML.
- If "expected" is unclear, the README and `.agents/pesudev-skills/auth/docs/api-contract.md` define it. If they are silent,
  decide from the code's intent and say so in the PR.

## 2. Write the regression test first

Put it in the suite that matches the layer (usually `tests/unit/`). Name it for the behaviour, e.g.
`test_empty_fields_list_is_a_400_not_a_500`. Run it and confirm it fails **for the bug's reason**.

## 3. Find the root cause

- Trace the path (`navigate-codebase`, reading order by task).
- `git log -L <start>,<end>:<file>` or `git blame` the lines; read the commit that introduced them.
  Many lines exist to prevent a specific failure described in their comment or commit.
- Ask what else shares the cause: the same pattern elsewhere (`rg -n "<pattern>" app`), the other
  exit paths of the same function (success, error, cancellation), the metric that should have
  recorded it.

## 4. Fix minimally

- Change the cause, not the symptom. Do not widen `except` clauses to hide it.
- Keep error semantics (`.agents/pesudev-skills/auth/docs/architecture.md`, error handling): 4xx for caller mistakes, 502 for
  upstream, 422 for profile parse, 500 only for our bugs.
- If the fix changes observable behaviour a caller could rely on, it may be breaking
  (`.agents/pesudev-skills/auth/docs/api-contract.md`); escalate if so.
- Add a why-comment if the fix is not self-evident.

## 5. Verify

- The regression test passes; the whole suite passes (`run-tests`).
- Related paths have tests too (for example the cancellation path of the same function).
- Metrics still satisfy the invariants in `.agents/pesudev-skills/auth/docs/metrics.md` if the bug touched request handling.
- For client lifecycle bugs, also follow `change-upstream-client`.

## 6. Document

If the README or Swagger described the buggy behaviour, fix them (`update-docs`). The commit is
`fix: <what is now right>`, with a body explaining the cause, the failure scenario and why the fix is
correct.
