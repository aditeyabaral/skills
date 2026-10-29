# Security

pesu-auth handles university credentials for every caller. A leak here is a leak of students'
PESU accounts. Treat every change to request handling, logging, upstream code or response parsing as
security relevant.

## What is sensitive

| Data                                                                                        | Where it appears                                      | Rule                                                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Passwords                                                                                   | request body, the login request                       | Never logged, stored, echoed, or put in an exception message, a `detail` or a metric                                                                                                                             |
| PESU's access token                                                                         | login response, the profile request's `Authorization` | Never logged or returned. `LoginResponse.access_token` has `repr=False`; never log the parsed login or the request headers                                                                                       |
| Usernames (SRN/PRN/email/phone)                                                             | logs (`user=...`), exception `detail`                 | Logged at INFO by design; do not add new places beyond logs                                                                                                                                                      |
| Profile data (name, email, mobile, gender, date of birth, ...)                              | response, logs                                        | Logged at INFO by design, so a login can be traced to what PESU returned: the profile (`app/pesu.py`) and the auth result (`app/app.py`). Every field is treated alike. Not a defect; do not remove or redact it |
| The rest of PESU's responses (photo, blood group, addresses, parents' details, marks, CGPA) | upstream responses only                               | Never kept: `app/models/upstream.py` declares only the fields the service returns, so the rest is dropped while parsing. Never add a field the API does not return                                               |
| `METRICS_TOKEN`                                                                             | environment                                           | Never logged; compared with `secrets.compare_digest` on bytes                                                                                                                                                    |
| `TEST_*` credentials and profile values                                                     | `.env`, CI secrets                                    | Never printed, committed, pasted into PRs, issues or logs, and never given to Copilot's cloud agent (the repo's `copilot` environment or `copilot-setup-steps.yml`)                                              |

## Threats and the controls that address them

- **Credential exposure in logs.** The validation handler logs only `type`, `loc`, `msg` of each
  error, never `input` (which is the whole body for a missing field). Any new handler or log line must
  follow the same rule. Test it with `caplog`: assert the password string is absent.
- **Personal data in parse errors.** A pydantic `ValidationError` quotes its input, which for an
  upstream response is personal data. It is never logged or chained: log
  `_validation_failure_summary(e)` and raise with `from None`. Unexpected fields are logged by PESU's
  name for them (`STUDENT_INFO.SRN`), never by value.
- **Log injection.** The username is logged, so `RequestModel` rejects one with line breaks or other
  invisible characters, and the one value from PESU logged on its own (an unknown campus name) is
  logged with `!r`. Keep both.
- **Session mixing.** Each login uses its own `httpx2.AsyncClient`, created for it and closed
  afterwards. Never share a client, a token or any state between requests.
- **Credential redirection.** Redirects are not followed, so a redirect in front of PESU cannot send
  the login form, password included, anywhere else.
- **Timing attacks on the metrics token.** `secrets.compare_digest` on bytes (header latin-1,
  configured token utf-8 surrogateescape). Do not replace with `==`, and do not compare `str`s
  (non-ASCII input would raise and become a 500).
- **Metric label cardinality (memory DoS).** Labels are clamped to closed sets; see
  `.agents/pesudev-skills/auth/docs/metrics.md`.
- **Mass assignment / unexpected input.** `RequestModel` is strict with `extra="forbid"` and length
  limits on both strings.
- **Information disclosure in errors.** Every error's `message` is fixed text; the specifics go in
  `detail`, which only the log sees. 500s return a generic message. PESU's own error text is never
  forwarded to the caller, and no error echoes the path or body the caller sent.
- **Browser-side issues.** Every response carries `X-Content-Type-Options: nosniff`,
  `Referrer-Policy: no-referrer` and `X-Frame-Options: DENY`; `/authenticate` and `/health`
  also carry `Cache-Control: no-store`. The API Explorer never stores a token in browser storage.
- **Resource exhaustion.** Upstream timeout is 10s; clients are always closed.
- **Container.** The image runs as the unprivileged UID 10001 with root-owned, read-only application
  files; CI checks the UID.
- **Supply chain.** Dependencies are pinned in `uv.lock`, and the Docker build and CI install with
  `uv sync --locked`, which also refuses a lock that no longer matches `pyproject.toml`.

## Review checklist for any change

- No new log line, exception message, `detail`, metric label or response field contains a password,
  a token or personal data the service does not already log.
- No new field read from PESU that the API does not return.
- No new unbounded label, unbounded cache, or unbounded loop against upstream.
- New inputs are validated by a strict pydantic model; unknown keys rejected.
- New endpoints: is it safe to leave unauthenticated? `/health` must stay open; anything exposing
  operational data should follow the `METRICS_TOKEN` pattern.
- New dependencies: maintained, widely used, license compatible with MIT, pinned via `uv lock`.
- Secrets are read from the environment, never hard-coded, and never from `.env` in app code (the
  app does not read `.env`).

## Auditing dependencies

```bash
uvx --python 3.14 pip-audit --strict --require-hashes --disable-pip -r <(uv export --frozen --no-dev --no-emit-project)
```

`uv export --frozen` only exports the lock, so `--frozen` is right here. Report findings with the
advisory ID and the fixed version; upgrading follows the `update-dependencies` skill.

## Reporting

Vulnerabilities are reported privately to maintainers (`.github/SECURITY.md`), never in a public
issue or PR. An agent that finds a vulnerability stops, writes the finding in its report to the
human, and does not open a public PR describing it.
