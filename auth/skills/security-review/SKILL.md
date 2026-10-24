---
name: security-review
description: Security and privacy review of a pesu-auth change - credential and personal-data exposure in logs, errors, metrics and responses, session isolation, input validation, token comparison, label-cardinality DoS, upstream abuse and dependency vulnerabilities - with tests proving each concern. Use for every change to request handling, logging, auth, upstream code or dependencies.
---

# Security review

Threat model and controls: `.agents/pesudev-skills/auth/docs/security.md`. pesu-auth sees students'
PESU passwords on every request; the bar is "cannot leak, even by accident".

## 1. Map the data flows in the diff

For every changed line that reads a request, a header, an upstream response, or writes a log, an
exception message, a metric label, a response field or a file, note what data can flow through it.

## 2. Check each flow

| Question                                                            | How to check                                                                                                                                          |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Can a password reach a log, exception message, metric or response?  | Follow `password` and the body from `RequestModel` to every sink, `detail=` included; check `logging.*` calls and f-strings in the diff               |
| Is any new personal data logged?                                    | Search 1 below; the INFO username and profile logs are deliberate (`.agents/pesudev-skills/auth/docs/security.md`), not findings                      |
| Are tokens (PESU's access token, METRICS_TOKEN) logged or returned? | Same search; never log the parsed login, the profile request's headers or a token                                                                     |
| Does more of PESU's response get kept?                              | No new field in `app/models/upstream.py` that the API does not return; a `ValidationError` from PESU's response never logged or chained (`from None`) |
| Can a caller write lines into the log?                              | Logged caller input stays printable-only (`RequestModel.validate_username`); logged upstream text uses `!r`                                           |
| Could two callers share a session?                                  | One client per login, created in `authenticate` and closed in its `finally`; nothing kept on `self` (`change-upstream-client`)                        |
| Is new input validated strictly?                                    | Pydantic model with `strict=True`, `extra="forbid"`; lengths or formats constrained where it matters                                                  |
| Can a caller create unbounded metric series or memory growth?       | Labels from closed sets only; no per-user or per-path caches                                                                                          |
| Can a caller make us hammer PESU?                                   | No retries in loops, no fan-out per request                                                                                                           |
| Secret comparison                                                   | `secrets.compare_digest` on bytes, as in `app/metrics/auth.py`                                                                                        |
| Error detail leakage                                                | Fixed `message`, specifics only in `detail`; 500s generic; upstream text not forwarded to callers                                                     |
| New endpoint exposure                                               | Should it be open? Operational data should use the `METRICS_TOKEN` pattern; `/health` stays open                                                      |
| Secrets in code, tests, fixtures, docs, commits                     | Search 2 below; read every hit                                                                                                                        |
| Does an agent gain secrets or write access?                         | `copilot-setup-steps.yml` sets no `env`/`secrets` and keeps `permissions: contents: read`; read-only roles' `tools` never gain `edit`                 |

Searches over the diff:

```bash
git diff upstream/dev...HEAD | rg -n "logging\.(info|warning|error|exception|debug)"   # 1. new log calls
git diff upstream/dev...HEAD | rg -in "password|token|secret|PES[12][0-9]{3}"           # 2. secrets and PRNs
```

## 3. Prove it

Write tests for the risky flows, for example:

```python
def test_validation_errors_do_not_log_the_password(client, caplog):
    with caplog.at_level("DEBUG"):
        client.post("/authenticate", json={"username": "u", "password": "hunter2", "profile": "yes"})
    assert "hunter2" not in caplog.text
```

## 4. Dependencies

If `pyproject.toml` or `uv.lock` changed:

```bash
uvx --python 3.14 pip-audit --strict --require-hashes --disable-pip -r <(uv export --frozen --no-dev --no-emit-project)
```

Check new packages for maintenance, popularity, license (MIT compatible) and install-time code.

## 5. Report

Findings with severity (`critical|high|medium|low`), the data or abuse involved, the failure
scenario and the fix. If you find an exploitable vulnerability (in the change or existing code),
**do not** put details in a commit, PR, issue or CI log: report it only to the human and point to
`.github/SECURITY.md`.
